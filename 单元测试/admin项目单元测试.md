# study-note
自己的学习笔记

#### phpunit 官方文档

```
https://phpunit.de/manual/6.5/en/appendixes.configuration.html#appendixes.configuration.logging
```



#### 修改 xml 文件内容

```

$xml = simplexml_load_file('phpunit.xml');

var_dump(((array)$xml->filter->whitelist->directory)[0]); die;

```

##### 转换为 json

```
<?php
function XML2JSON($xml) {

        function normalizeSimpleXML($obj, &$result) {
            $data = $obj;
            if (is_object($data)) {
                $data = get_object_vars($data);
            }
            if (is_array($data)) {
                foreach ($data as $key => $value) {
                    $res = null;
                    normalizeSimpleXML($value, $res);
                    if (($key == '@attributes') && ($key)) {
                        $result = $res;
                    } else {
                        $result[$key] = $res;
                    }
                }
            } else {
                $result = $data;
            }
        }
        normalizeSimpleXML(simplexml_load_string($xml), $result);
        return json_encode($result);
    }
?>
```



#### 执行命令

##### 命令行，执行 目录下所有 *Test 文件，不符合命名 则不会计算覆盖率

```
ps: docker exec -it phpunit bash 
在项目根目录下 执行 

./vendor/bin/phpunit

测试目录
admin.s.weibo.com/test/app/Service/IndexTest.php

查看覆盖率文件地址
/data1/apache2/htdocs/tmp/
```



##### 参考文档

```
https://gist.github.com/Zhwt/905404fea48a8d3e70823d4f6421d3a5
```

##### admin 项目，根目录 phpunit.xml 文件内容

```
<phpunit bootstrap="/data1/apache2/htdocs/admin.s.weibo.com/test/Ainit.php"
                colors="true"
                 stopOnFailure="false"
                 processIsolation="false"
                timeoutForSmallTests="1"
                timeoutForMediumTests="10"
                timeoutForLargeTests="60"
                verbose="false">
        <testsuites>
                <testsuite name="My Test Suites">
                        <directory>test</directory>
                </testsuite>
        </testsuites>


        <filter>
                <whitelist addUncoveredFilesFromWhitelist="false">
<!--                        <directory suffix=".php">/data1/apache2/htdocs/admin.s.weibo.com/test/app/Service/</directory>-->
<!--                        <file>/data1/apache2/htdocs/admin.s.weibo.com/test/app/Service/IndexTest</file>-->
<!--                        <directory suffix=".php">/data1/apache2/htdocs/admin.s.weibo.com/application/common/service/huati/</directory>-->
                        <file>/data1/apache2/htdocs/admin.s.weibo.com/application/common/service/huati/QueryShieldService.php</file>
                        <file>/data1/apache2/htdocs/admin.s.weibo.com/application/common/service/hotRank/RankSnap.php</file>

<!--                        <exclude>-->
<!--                                <directory suffix=".php">/data1/apache2/htdocs/admin.s.weibo.com/application/common/service/huati</directory>-->
<!--                                <file>/data1/apache2/htdocs/admin.s.weibo.com/application/common/service/huati/QueryManageService.php</file>-->
<!--                        </exclude>-->
                </whitelist>
        </filter>

        <logging>
                <log type="coverage-html" target="/data1/apache2/htdocs/tmp/" lowUpperBound="35"
                     highLowerBound="70"/>
                <log type="coverage-clover" target="/data1/apache2/htdocs/tmp/coverage.xml"/>
<!--                <log type="coverage-php" target="/data1/apache2/htdocs/tmp//coverage.serialized"/>-->
                <log type="coverage-text" target="php://stdout" showUncoveredFiles="false"/>

                <!--                <log type="junit" target="/data1/apache2/htdocs/tmp//logfile.xml" logIncompleteSkipped="false"/>-->
<!--                <log type="testdox-html" target="/data1/apache2/htdocs/tmp//testdox.html"/>-->
<!--                <log type="testdox-text" target="/data1/apache2/htdocs/tmp/testdox.txt"/>-->
        </logging>

</phpunit>


//whitelist.directory|file 两个是并列关系，可以引入文件夹 或者 指定文件

```

##### 实现方法函数 mock

```
<?php
use PHPUnit\Framework\TestCase;
use app\common\service\huati\QueryShieldService;
use app\common\model\huati\TopicShieldModel;
use app\common\helper\Mc;
class IndexTest extends TestCase
{

    private $TopicShieldModel ;
    private $QueryShieldService ;
    public function setUp(){
        $this->TopicShieldModel = $this->getMockBuilder(TopicShieldModel::class)->setMethods(['edit','insertAll'])->getMock();
        $this->QueryShieldService =$this->getMockBuilder(QueryShieldService::class)->setMethods(['addChangeQueue'])->getMock();
    }
    public  function SomethingIsTrue(){

        return [
            [
                [
                    'topic' => '',
                ],
                []
            ],
            [
                [
                    'topic' => '我要报bug',
                    'shield_words' => '',
                ],
                true
            ]
            ,
            [
                [
                    'topic' => '我要报bug',
                    'shield_words' => '我是屏蔽词',
                ],
                true
            ]
            ,
        ];
    }
    /**
     * @dataProvider SomethingIsTrue
     */
    public function testSomethingIsTrue($params, $equals)
    {

        // 配置桩件。
         $this->TopicShieldModel->method('edit')->willReturn(true);
         $this->TopicShieldModel->method('insertAll')->willReturn(true);

        $this->QueryShieldService->method('addChangeQueue')->willReturn(true);


        $QueryShieldService = $this->QueryShieldService;
        // 配置桩件。

//        $QueryShieldService = new QueryShieldService();
        $QueryShieldService->TopicShieldModel = $this->TopicShieldModel;

        $res = $QueryShieldService->add($params);
        $this->assertEquals($equals, $res);

    }

}
```

