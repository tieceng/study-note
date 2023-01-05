# study-note
自己的学习笔记



```
<?php
use PHPUnit\Framework\TestCase;

class StubTest extends TestCase
{
    public function testStub()
    {
        // 为 SomeClass 类创建桩件。
        $stub = $this->createMock(SomeClass::class);

        // 配置桩件。
       $stub->expects($this->any())->method('doSomething')->willReturn('foo');。

        // 现在调用 $stub->doSomething() 将返回 'foo'。
        $this->assertEquals('foo', $stub->doSomething());
    }
}
?>
```

