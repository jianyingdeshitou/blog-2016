# 学习使用PHP_CodeSniffer（二）

> http://blog.csdn.net/xinhaozheng/article/details/3334583

## PHP_CodeSniffer所使用的编码标准

`PHP_CodeSniffer`中的编码标准是指一系列sniff文件的集合，每个sniff文件对应某个编码标准的特定部分（如代码对齐，或注释）。`PHP_CodeSniffer`中可以同时安装有多个编码标准，也就是说你可以用`PHP_CodeSniffer`对多个项目应用各自的编码标准进行检查。

用下列命令可以查看`PHP_CodeSniffer`中已经安装的编码标准：

```shell
$ phpcs -i
The installed coding standards are PHPCS, Squiz, MySource, PEAR and Zend
```

`PHP_CodeSniffer`中的编码标准都在`XXX/CodeSniffer/Standards`目录下各自有一个以标准名称命名的独立目录，如现在`PHP_CodeSniffer`安装目录下已经安有PHPCS，Squiz，MySource，PEAR，Zend目录。每个目录下都包含了sniff文件目录，其中包含了用来定义编码标准各个部分的sniff文件，也可以用子目录将sniff文件分类存放。所以可以使用下列命令在`PHP_CodeSniffer`创建一个名为Seagull的编码标准的目录结构：

```shell
$ cd XXX/CodeSniffer/Standards
$ mkdir Seagull
$ mkdir Seagull/Sniffs
```

除此之外，我们还需要在Seagull目录下创建一个类文件，向`PHP_CodeSniffer`说明关于这个标准的基本信息并告诉`PHP_CodeSniffer`这个目录中包含这个标准的sniffs文件。这个类文件必须以此编码标准的名称为前缀，`CodingStandard`为后缀：

```shell
$ touch Seagull/SeagullCodingStandard.php
```

内容应该如下：

```php
    <?php
    /**
     * Seagull Coding Standard.
     *
     * PHP version 5
     *
     * @category  PHP
     * @package   PHP_CodeSniffer
     * @author    Your Name <you@domain.net>
     * @license   http://matrix.squiz.net/developer/tools/php_cs/licence BSD Licence
     * @version   CVS: $Id: coding-standard-tutorial.xml,v 1.6 2007/10/15 03:28:17 squiz Exp $
     * @link      http://pear.php.net/package/PHP_CodeSniffer
     */
    require_once 'PHP/CodeSniffer/Standards/CodingStandard.php';
    /**
     * Seagull Coding Standard.
     *
     * @category  PHP
     * @package   PHP_CodeSniffer
     * @author    Your Name <you@domain.net>
     * @license   http://matrix.squiz.net/developer/tools/php_cs/licence BSD Licence
     * @version   Release: @package_version@
     * @link      http://pear.php.net/package/PHP_CodeSniffer
     */
    class PHP_CodeSniffer_Standards_Seagull_SeagullCodingStandard extends PHP_CodeSniffer_Standards_CodingStandard
    {
    }//end class
    ?>
```

> 注：上面的这个代码示例中类的实体是空的。你也可以添加方法，详细的后面会介绍。

这样你就可以往Seagull编码标准的`sniff`文件目录中添加自己定义的sniff文件了，当然为了查找方便，你可以在`Sniff`文件目录中创建子目录将不同目的的sniff文件分门别类。

## 创建sniff文件

每个sniff对应一个PHP文件，文件命名最好是描述该sniff所强调的编码标准格式，并以`Sniff.php`结尾。比如我们不想让代码中使用`#`注释，我们可以创建一个名为`DisallowHashCommentsSniff.php`，并将它归入到`Commenting`子目录下。


```shell
$ cd XXX/CodeSniffer/Standards/Seagull/Sniffs
$ mkdir Commenting
$ touch Commenting/DisallowHashCommentsSniff.php
```

每个sniff类必须实现`PHP_CodeSniffer_Sniff`接口，这样在调用时`PHP_CodeSniffer`才会创建一个sniff实例，`PHP_CodeSniffer_Sniff`定义两个在sniff文件中的类必须要实现的方法：`register`和`process`。

sniff通过调用`register`告诉`PHP_CodeSniffer`它要检查编码标准哪些方面（也就是这个sniff要处理或者说会处理哪些类型的`token`)。这样当`PHP_CodeSniffer`在检查代码文件时碰到这些`token`时就会调用`process`方法来处理，同时付给`process`两个参数，第一个是`PHP_CodeSniffer_File` 对象（即当前正在被处理的代码文件），另一个是出现错误的`token`在`token`堆栈中的索引。

什么是`token`堆栈？我的认识是类似于php在解析php代码时会先将php代码分解成一个`token`数组。在sniff文件中通过调用`PHP_CodeSniffer_File`对象的`getTokens()`可以获取一个`token`数组，数组元素是以每个`token`在堆栈中的位置为索引。所有的`token`对应一个数组，包含`code`，`type`，`content`索引对应的元素。其中`code`索引对应的元素是这个`token`类型对应的唯一整数值，`type`则对应`token`类型的字符串名称，`type`还以一个与此字符串同名的全局常量。`content`索引的值是这个token所对应的代码。

> 注：有些`token`可能拥有更多的的元素。可以查看`PHP/CodeSniffer/File.php`中的类类注释有一个`token`列表和相应的索引说明。

这样我们就可以写一个自己的`sniff`文件了，先通过`register`向`PHP_CodeSniffer`注册此`sniff`要检查的`token`类型，然后在`process`里按照自已需要的编码标准进行处理，如果有错误发生就可以调用`PHP_CodeSniffer_File` 对象的`addError`来指示出错，其实也就是在`PHP_CodeSniffer`的检查结果中增加一条错误信息。当然如果错误不是太严重，你可以调用`addWarning`来输出一个警告信息。

下面代码示例了不允许在代码中使用类似`PERL`的`#`开头的单选注释：

```php
    <?php
    /**
     * PHP_CodeSniffer tokenises PHP code and detects violations of a
     * defined set of coding standards.
     *
     * PHP version 5
     *
     * @category  PHP
     * @package   PHP_CodeSniffer
     * @author    Your Name <you@domain.net>
     * @license   http://matrix.squiz.net/developer/tools/php_cs/licence BSD Licence
     * @version   CVS: $Id: coding-standard-tutorial.xml,v 1.6 2007/10/15 03:28:17 squiz Exp $
     * @link      http://pear.php.net/package/PHP_CodeSniffer
     */

    require_once 'PHP/CodeSniffer/Sniff.php';

    /**
     * This sniff prohibits the use of Perl style hash comments.
     *
     * An example of a hash comment is:
     *
     * <code>
     *  # This is a hash comment, which is prohibited.
     *  $hello = 'hello';
     * </code>
     *
     * @category  PHP
     * @package   PHP_CodeSniffer
     * @author    Your Name <you@domain.net>
     * @license   http://matrix.squiz.net/developer/tools/php_cs/licence BSD Licence
     * @version   Release: @package_version@
     * @link      http://pear.php.net/package/PHP_CodeSniffer
     */
    class Seagull_Sniffs_Commenting_DisallowHashCommentsSniff implements PHP_CodeSniffer_Sniff
    {


        /**
         * Returns the token types that this sniff is interested in.
         *
         * @return array(int)
         */
        public function register()
        {
            return array(T_COMMENT);

        }//end register()


        /**
         * Processes the tokens that this sniff is interested in.
         *
         * @param PHP_CodeSniffer_File $phpcsFile The file where the token was found.
         * @param int                  $stackPtr  The position in the stack where
         *                                        the token was found.
         *
         * @return void
         */
        public function process(PHP_CodeSniffer_File $phpcsFile, $stackPtr)
        {
            $tokens = $phpcsFile->getTokens();
            if ($tokens[$stackPtr]['content']{0} === '#') {
                $error = 'Hash comments are prohibited';
                $phpcsFile->addError($error, $stackPtr);
            }

        }//end process()


    }//end class

    ?>
```

## Coding Standard类方法

正如之前所提到的，每个编码标准都有一个`Coding Standard`类，`PHP_CodeSniffer` 通过这个类获取关于这个编码标准的信息，同时这个类也向`PHP_CodeSniffer`标识了它亿在的这个文件目录中包含有sniff文件。我们可以通过重写这个类的两个方法来提供关于这个编码标准的额外信息。

* getIncludedSniffs()

这个方法可以告诉`PHP_CodeSniffer`除了这个编码对应的`sniffs`文件之外，你还要加载来自其它编码标准的`sniff`文件。它可以包含单个`sniff`文件，也可以包含整个目录或整个编码标准对应的`sniff`文件。

看如下示例：

```php
    <?php
    /**
     * Return a list of external sniffs to include with this standard.
     *
     * The MyStandard coding standard uses some generic sniffs, and
     * the entire PEAR coding standard.
     *
     * @return array
     */
    public function getIncludedSniffs()
    {
        return array(
                'PEAR',
                'Generic/Sniffs/Formatting/MultipleStatementAlignmentSniff.php',
                'Generic/Sniffs/Functions',
               );
    }//end getIncludedSniffs()
    ?>
```

> 注：在用此方法包含另外一个编码标准的的sniff文件时，如果这个标准本身又有包含其它编码标准的sniff文件时，这些sniff文件也会被包含进来。所以，基于一个编码标准创建一个新的编码标准是非常方便的。

* getExcludedSniffs()

这个方法的作用和上面的方法相反。看下面的示例，先是包含了某个整个的编码标准的sniff文件，但是又不需要其中的某个sniff文件。

```php
    <?php
    /**
     * Return a list of external sniffs to include with this standard.
     *
     * The MyStandard coding standard uses all PEAR sniffs except one.
     *
     * @return array
     */
    public function getIncludedSniffs()
    {
        return array(
                'PEAR',
               );
    }//end getIncludedSniffs()
    /**
     * Return a list of external sniffs to exclude from this standard.
     *
     * The MyStandard coding standard uses all PEAR sniffs except one.
     *
     * @return array
     */
    public function getExcludedSniffs()
    {
        return array(
                'PEAR/Sniffs/ControlStructures/ControlSignatureSniff.php',
               );
    }//end getExcludedSniffs()
    ?>
```

（待续）
