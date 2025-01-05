# PHP单元测试PHPUnit

## 简介

\# PHPUnit的依赖

* [dom](http://php.net/manual/en/dom.setup.php)，PHP默认启用
* [json](http://php.net/manual/en/json.installation.php)，PHP默认启用
* [pcre](http://php.net/manual/en/pcre.installation.php)，PHP默认启用
* [reflection](http://php.net/manual/en/reflection.installation.php)，PHP默认启用
* [spl](http://php.net/manual/en/spl.installation.php)，PHP默认启用
* [tokenizer](http://php.net/manual/en/tokenizer.installation.php)，PHP默认启用，生成代码覆盖率测试报告用到
* [xdebug](https://xdebug.org/)，需要自己安装，生成代码覆盖率测试报告用到
* [xmlwriter](http://php.net/manual/en/xmlwriter.installation.php)，PHP默认启用，生成XML格式的报告用到

检查PHP是否启用了这些模块可以使用如下命令

```bash
php -m | grep '模块名称'
```

## PHPUnit的安装

### PHP档案包方式

```bash
wget https://phar.phpunit.de/phpunit.phar #  下载档案包
chmod +x phpunit.phar #  赋可执行权限
sudo mv phpunit.phar /usr/local/bin/phpunit
phpunit --version #  查看PHPUnit版本
```

### Composer方式

Composer使用参考https://getcomposer.org/

composer文件如下

```json
{
    "require-dev": {
        "phpunit/phpunit": "5.0.*"
    }
}
```

## 编写PHPUnit测试

### 惯例和基本步骤

1. 针对类Class的测试写在类ClassTest中。
2. ClassTest通常继承自PHPUnit\_Framework\_TestCase。
3. 测试都是命名为test\*的公用方法（也可以在放的文档注释块中使用@test标注将其标记未测试方法）。
4. 在测试方法内，类似于assertEquals()这样的断言方法用来对实际值与预期值的匹配做出断言

举个例子：

```php
<?php
class HelloWolrdTest extends PHPUnit_Framework_TestCase
{
    public function testString()
    {
        $string = 'Hello, World';
        $this->assertEquals('Hello, World', $string);
        $this->assertNotEquals('Hello,World', $string);
    }

    public function testAarry()
    {
        $arr = array();
        $this->assertEmpty($arr);
        //$this->assertArrayHasKey('hello', $arr);

        $arr['hello'] = 'world';
        //$this->assertEmpty($arr);
        $this->assertArrayHasKey('hello', $arr);
    }
}
```

执行测试命令

```bash
phpunit HelloWorldTest
```

结果

```bash
PHPUnit 4.8.24 by Sebastian Bergmann and contributors.

..

Time: 103 ms, Memory: 12.50Mb

OK (2 tests, 4 assertions)
```

### 依赖

@depends表示依赖关系

```php
<?php class DependenceTest extends PHPUnit_Framework_TestCase
{
    public function testEmpty()
    {
        $arr = array();
        $this->assertEmpty($arr);

        return $arr;
    }

    /**
     * @depends testEmpty
     */
    public function testPush(array $arr)
    {
        array_push($arr, 'foo');
        $this->assertEquals('foo', $arr[count($arr) - 1]);
        $this->assertNotEmpty($arr);

        return $arr;
    }

    /**
     * @depends testPush
     */
    public function testPop(array $arr)
    {
        $this->assertEquals('foo', array_pop($arr));
        $this->assertEmpty($arr);
    }

    public function testOne()
    {
        $this->assertTrue(false);
    }

    /*
     * @depends testOne
     */
    public function testTwo()
    {
        $this->assertTrue(1);
    }

    public function testProducerFirst()
    {
        $this->assertTrue(true);
        return 'first';
    }

    public function testProducerSecond()
    {
        $this->assertTrue(true);
        return 'second';
    }

    /**
     * @depends testProducerFirst
     * @depends testProducerSecond
     */
    public function testOneConsumer()
    {
        $this->assertEquals(
            array('first', 'second'),
            func_get_args()
        );
    }

    /**
     * @depends testProducerSecond
     * @depends testProducerFirst
     */
    public function testTwoConsumer()
    {
        $this->assertEquals(
            array('first', 'second'),
            func_get_args()
        );
    }
}

```

### 数据提供器

@dataProvider表示数据提供器

```php
<?php

class DataProviderTest extends PHPUnit_Framework_TestCase
{
    public function getTestAddData()
    {
        return array(
            array(0, 0, 0),
            array(0, 1, 1),
            array(1, 1, 2),
            array(2, 2, 5),
        );
    }

    /**
     * @dataProvider getTestAddData
     */
    public function testAdd($op1, $op2, $sum)
    {
        $this->assertEquals($sum, $op1 + $op2);
    }

    /**
     * @dataProvider getTestMulData
     */
    public function testMul($op1, $op2, $mul)
    {
        $this->assertEquals($mul, $op1 * $op2);
    }

    public function getTestMulData()
    {
        return array(
            array(1, 1, 1),
            array(2, 3, 6),
            array(3, 3, 8),
        );
    }
}

```

### 对异常进行测试

@expectedException表示程序抛出异常

@expectedException可以结合@expectedExceptionCode、@expectedExceptionMessage、@expectedExceptionMessageRegExp使用

```php
<?php

class ExceptionTest extends PHPUnit_Framework_TestCase
{
    /**
     * @expectedException InvalidArgumentException
     */
    public function testException()
    {
        throw new InvalidArgumentException('', 0);
        //throw new Exception('', 0);
    }

    /**
     * @expectedException Exception
     * @expectedExceptionCode 404
     */
    public function testExceptionCode()
    {
        throw new Exception('exception', 404);
        //throw new Exception('exception', 30);
    }

    /**
     * @expectedException Exception
     * @expectedExceptionMessage test
     */
    public function testExceptionMessage()
    {
        throw new Exception('test', 0);
        //throw new Exception('abcdefg', 0);
    }

    /**
     * @expectedException Exception
     * @expectedExceptionMessageRegExp /[a-z]+/
     */
    public function testExceptionMessageRegExp()
    {
        throw new Exception('test', 0);
        //throw new Exception('TEST', 0);
    }

    public function testOne()
    {
        $this->assertEquals(1, 1);
    }
}
```

### 对输出进行测试

expectOutputString方法可以测试程序的输出

```php
<?php

class OutputTest extends PHPUnit_Framework_TestCase
{
    public function testOutputOne()
    {
        $this->expectOutputString('foo');
        print 'test';
    }

    public function testOutputTwo()
    {
        $this->expectOutputString('foo');
        print 'foo';
    }
}
```

## PHPUnit的输出

先看一个demo

```php
<?php

class SummaryTest extends PHPUnit_Framework_TestCase
{

    public function testSuccess()
    {
        $this->assertEquals(1, 1);
    }

    public function testFail()
    {
        $this->assertEquals(2, 1);
    }

    public function testException()
    {
        throw new Exception('', 0);
    }

    /*
     * @depends testThree
     */
    public function testSkip()
    {
        $this->assertEquals(2, 1);
    }
}
```

执行命令

```bash
phpunit SummaryTest
```

输出结果如下

```bash
PHPUnit 4.8.24 by Sebastian Bergmann and contributors.

.FEF

Time: 96 ms, Memory: 12.00Mb

There was 1 error:

1) SummaryTest::testException
Exception:

/Users/kenneth/codes/php/phpunit/SummaryTest.php:18

--

There were 2 failures:

1) SummaryTest::testFail
Failed asserting that 1 matches expected 2.

/Users/kenneth/codes/php/phpunit/SummaryTest.php:13

2) SummaryTest::testSkip
Failed asserting that 1 matches expected 2.

/Users/kenneth/codes/php/phpunit/SummaryTest.php:26

FAILURES!
Tests: 4, Assertions: 3, Errors: 1, Failures: 2.
```

看输出第二行，有.FEF等，分表表示如下信息：

1. . ：表示断言通过
2. F：表示断言失败
3. E：表示测试过程中抛出一个错误
4. R：表示测试被标记为有风险
5. S：表示测试被跳过
6. I ：表示测试被标记未不完整或未实现
