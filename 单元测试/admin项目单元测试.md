# study-note
自己的学习笔记

#### 执行命令

##### 命令行，执行 目录下所有 *Test 文件，不符合命名 则不会计算覆盖率

```
在项目根目录下 执行 

./vendor/bin/phpunit

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
                        <directory suffix=".php">/data1/apache2/htdocs/admin.s.weibo.com/test/app/Service/</directory>
                        <file>/data1/apache2/htdocs/admin.s.weibo.com/test/app/Service/IndexTest</file>
                        <directory suffix=".php">/data1/apache2/htdocs/admin.s.weibo.com/application/common/service/huati/</directory>
                        <file>/data1/apache2/htdocs/admin.s.weibo.com/application/common/service/huati/QueryShieldService</file>
                </whitelist>
        </filter>

        <logging>
                <log type="coverage-html" target="/data1/apache2/htdocs/tmp/" lowUpperBound="35"
                     highLowerBound="70"/>
<!--                <log type="coverage-clover" target="/data1/apache2/htdocs/tmp/coverage.xml"/>-->
<!--                <log type="coverage-php" target="/data1/apache2/htdocs/tmp//coverage.serialized"/>-->
<!--                <log type="coverage-text" target="php://stdout" showUncoveredFiles="false"/>-->
<!--                <log type="junit" target="/data1/apache2/htdocs/tmp//logfile.xml" logIncompleteSkipped="false"/>-->
<!--                <log type="testdox-html" target="/data1/apache2/htdocs/tmp//testdox.html"/>-->
<!--                <log type="testdox-text" target="/data1/apache2/htdocs/tmp//testdox.txt"/>-->
        </logging>

</phpunit>

```

