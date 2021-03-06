---
title: Migrating from Flask-Script to the New Flask CLI
tag: [Python,Flask]
comments: true
date: 2017-10-19
---







>原文:https://blog.miguelgrinberg.com/post/migrating-from-flask-script-to-the-new-flask-cli

![](http://ww1.sinaimg.cn/large/006wYWbGly1fknogoype5j30ci0810so.jpg)

在Falsk 0.11版本, Flask 基于 [Click](http://click.pocoo.org/5/) 引进了一个新的命令行功能,该功能包含了 <code>flask</code>命令. 在此之前 Flask 并没有提供任何关于构建命令行接口(CLIs)的支持,但是 [Flask=Script](http://flask-script.readthedocs.io/en/latest/)作为三方拓展提供了类似的功能.

在 Flask CLI发布超过一年的时间里,我仍然看见有许多项目基于 Flask-Script. 我猜测这是由于没有特别重要的原因促使人们迁移到 Flask CLI中，毕竟 Flask-Script 运转地很好,至少没什么毛病. 但是实际上,自从 2014 年开始 Flask-Script 就再也没有过官方发布版本并且看起来也没有再维护了.在本文中，我想向你展示如何将 Flasky 应用程序从 Flask-Script 迁移到 Click ，以便你可以了解两者的不同之处，并决定是否迁移应用程序.

# Flask-Script 有什么问题？

在我看来,Flask-Script 有个严重的缺陷.但有趣的是，这是一个由Flask reloader 中的老的设计问题直接引起的问题,如果你使用app.run()启动应用程序,而不使用Flask-Script,仍然会出现这个问题.

如果你在调试模式下启动应用程序，则会运行两个 Flask 进程. 第一个进程用来观察源文件的更改，第二个是实际的 Flask 服务器. 当任何源文件更改时，监视进程将杀死服务器进程,然后启动另一个进程,从而使用新的源文件.

当你无意中在某个源文件中引入语法错误时，会出现一个问题. 这个时候观察者进程并不知道,所以它仍然会杀死旧的服务器,并启动一个新的. 但是新服务器不会启动,因为 Python 在解析修改后的文件时会引发错误.当服务器进程在错误中退出时,观察者进程也会放弃并退出,从而迫使你在修复代码中的错误后重新启动整个进程.

当您使用新的 Flask Run 命令启动重新加载程序时,效果会好得多.在发送第一个请求之前,不会导入应用程序,此时,如果在导入源文件时发生错误,就会像处理运行时发生的错误一样进行处理.这意味着,如果启用了基于 Web 的调试器,将在那里报告错误.新的观察者进程也更智能，如果服务器进程死了也不会退出.相反,它将继续监视源文件,并尝试在进行更多更改后再次启动服务器.

# 新的 Flask CLI

Flask CLI 比 Flask-Script 更好吗?让我们首先回顾一下所有 Flask 应用程序获得的默认命令，看看它们的性能如何.

当你安装Flask 0.11或更高版本时，您将在虚拟环境中安装一个flask命令:

```shell
(venv) $ flask
Usage: flask [OPTIONS] COMMAND [ARGS]...

  This shell command acts as general utility script for Flask applications.

  It loads the application configured (through the FLASK_APP environment
  variable) and then provides commands either provided by the application or
  Flask itself.

  The most useful commands are the "run" and "shell" command.

  Example usage:

    set FLASK_APP=hello.py
    set FLASK_DEBUG=1
    flask run

Options:
  --version  Show the flask version
  --help     Show this message and exit.

Commands:
  db     Perform database migrations.
  run    Runs a development server.
  shell  Runs a shell in the app context.
```

使用 Flask-script 时，你必须创建一个驱动程序脚本，通常称为 <code>manage.py</code>. 查看上面的内容，可以看到<code>./manage.py</code> <code>runserver</code>映射到<code>flask run</code>, <code>./manage.py shell</code> 映射到<code>flask shell</code>没什么区别对吧？

这里其实有一个令人困扰的问题. manage.py和flask命令在发现Flask应用程序实例的方式上有所不同. 对于Flask-Script，应用程序作为参数直接或以应用程序工厂函数的形式提供给Manager类. 但是，新的FlaskCLI期望应用程序实例在flask_app环境变量中提供，通常将其设置为定义它的模块的文件名.  Flask将在该模块中查找应用程序或应用程序对象，并将其用作应用程序.

不幸的是,在新的 Flask CLI 中没有对应用程序工厂功能的直接支持.使用工厂函数需要遵循的方法是定义一个调用工厂函数来创建app对象的模块,然后在 <code>flask_app</code>中引用该模块.这在概念上类似于 <code>Django</code> 应用程序中的 <code>wsgi.py</code> 模块.

如果你希望为[Flasky](https://github.com/miguelgrinberg/flasky)支持新的 Flask CLI，可以编写一个flasky.py文件，如下所示：

```python
import os
from app import create_app

app = create_app(os.getenv('FLASK_CONFIG') or 'default')
```

注意，我从manage.py复制了这些行，一旦完全迁移到新的CLI，我们就不会使用这些行.

# "flask run" 命令

添加<code>flasky.py</code>模块后，可以使用以下命令启动 Flask 开发服务器:

```shell
$ export FLASK_APP=flasky.py
$ flask run
```

如果你使用的是 Windows，则 <code>FLASK_APP</code>环境变量的设置方式略有不同:

```shell
> set FLASK_APP=flasky.py
> flask run
```

<code>flask run</code>命令具有启用或禁用重新加载程序和调试器的选项,还可以设置 IP 地址和端口,以便服务器侦听客户端请求. 这些选项与 Flask-script 中的选项并不相同,但它们比较接近.你可以使用<code>flask run --help</code>看见所有可用的选项.

正如我敢肯定，大家都知道，大多数 Flask 的应用程序有这行代码在主脚本的底部:

```python
if __name__ == '__main__':
    app.run()
```

当你不使用任何 CLI 支持时,这就是你实际如何启动服务器的方法.如果你将此保留在脚本中并开始使用新的 CLI ,不会导致任何问题，但实际上并不需要它，因此你可以删除它.

# "flask shell" 命令
<code>shell</code>命令基本上是相同的，但是在如何定义要自动导入到 shell 上下文中的其他符号方面有一点差异. 这是一个在处理应用程序时可以节省大量时间的特性.通常,你可以在 shell 的测试或调试会话中添加模型类,数据库实例和其他可能与之交互的对象.

对于 Flask-Script 来说, Flasky 应用有下述 shell 上下文的定义:

```python
def make_shell_context():
    return dict(app=app, db=db, User=User, Follow=Follow, Role=Role,
                Permission=Permission, Post=Post, Comment=Comment)
manager.add_command("shell", Shell(make_context=make_shell_context))
```

正如你在上面看到的, <code>make_shell_text()</code>函数在定义 <code>shell</code> 选项的 <code>add_command</code> 调用中被引用. Flask-Script 在启动shell会话之前调用此函数，并在其中包含字典中返回的所有符号.

Flask CLI提供了相同的功能,使用装饰器来标识提供 shell 上下文项的函数(实际上,可以有多个函数)。为了具有与上面相同的功能，我扩展了<code>flasky.py</code>模块:

```python
import os
from app import create_app, db
from app.models import User, Follow, Role, Permission, Post, Comment

app = create_app(os.getenv('FLASK_CONFIG') or 'default')

@app.shell_context_processor
def make_shell_context():
    return dict(app=app, db=db, User=User, Follow=Follow, Role=Role,
                Permission=Permission, Post=Post, Comment=Comment)
```

因此,如你所见,该函数是相同的,但需要<code>@app.Shell_Context_Processor</code>装饰器，以便 Flask 知道它.

# 来自 Flask Extensions 的命令

Flask-Script 成功的一个重要原因是它允许其他 Flask 扩展添加自己的命令.实际上,我已经利用了这个功能在我的[Flask-Migrate](https://github.com/miguelgrinberg/flask-migrate)扩展.

那么,如何将这些命令迁移到Click CLI呢?不幸的是，你只能依赖扩展作者为你进行迁移.如果你使用的任何扩展,与 Flask-Script 协同工作,并且还没有更新到与Flask CLI协同工作的话,那么你将非常不幸,你将需要继续使用 Flask-Script.至少在与相关扩展交互时是这样.如果你使用 Flask-Migrate,你不必担心,因为我已经更新了它以支持 Click 和 Flask-Script CLIs,确保您正在使用最新版本.

特别是对于Flask-Migrate,<code>./manage.py db....</code>.在 Flask-script 版本中, Flask-Migrate 扩展是在<code>manage.py</code>中初始化的.对于 Flask CLI 版本,我们可以将其移到新的<code>flasky.py</code>模块，我相信你现在已经意识到它正在成为 Flask-Script 的<code>manage.py</code>的替代品.唯一的区别是它不是直接执行的脚本.下面是添加了Flask-Migrate集成的模块:

```python
import os
from app import create_app, db
from app.models import User, Follow, Role, Permission, Post, Comment
from flask_migrate import Migrate

app = create_app(os.getenv('FLASK_CONFIG') or 'default')
migrate = Migrate(app, db)

@app.shell_context_processor
def make_shell_context():
    return dict(app=app, db=db, User=User, Follow=Follow, Role=Role,
                Permission=Permission, Post=Post, Comment=Comment)
```

作为例子,你可以使用以下命令将 Flasky 数据库升级到最新的修订版本,但请记住,需要设置<code>FLASK_APP</code>才能使其工作:

```shell
$ flask db upgrade
```

# 应用命令

Flask-Script 的另一个不错的特性是，它允许您将自定义任务编写为函数,然后通过添加装饰器将其自动转换为 CLI 中的命令.在 Flasky 中,我使用这个特性添加了一些命令.例如,下面是我如何实现 Flask-script 的<code>./manage.py test</code>命令:

```python
@manager.command
def test(coverage=False):
  """Run the unit tests."""
  # ...
```

<code>@manager.command</code>装饰器是将函数公开为 <code>test</code>命令所需的,它甚至可以检测到<code>coverage</code>参数,并将其添加为 --coverage 选项.

基于装饰器的命令实际上是 Click 的面包和黄油,所以这是可以毫无问题地迁移的.Flask CLI 的等效函数可以放在 <code>flasky.py</code>中,如下所示:

```python
import click

# ...

@app.cli.command()
@click.option('--coverage/--no-coverage', default=False, help='Enable code coverage')
def test(coverage):
    """Run the unit tests."""
    # ...
```

<code>@app.cli.command()</code>装饰器为 Click 提供了一个接口.在Flask-script中,命令是在安装了应用程序上下文的情况下执行的,但是在使用 Flask CLI 的情况下,如果你不需要的话,你可以禁用它. 函数的实际代码不需要更改,但请注意,<code>--coverage</code>选项需要使用<code>@click.OptionDecorator</code>显式地给出,这与Flask-Script不同,后者的选项自动从函数参数列表派生.

另外两个在我的 Flasky 的应用程序中的自定义函数也可以采用相同的方法. <code>flasky.py</code>模块的完整实现如下:

```python
import os
from app import create_app, db
from app.models import User, Follow, Role, Permission, Post, Comment
from flask_migrate import Migrate
import click

app = create_app(os.getenv('FLASK_CONFIG') or 'default')
migrate = Migrate(app, db)

@app.shell_context_processor
def make_shell_context():
    return dict(app=app, db=db, User=User, Follow=Follow, Role=Role,
                Permission=Permission, Post=Post, Comment=Comment)

@app.cli.command()
@click.option('--coverage/--no-coverage', default=False, help='aaa')
def test(coverage=False):
    "Test coverage"
    # ...

@app.cli.command()
@click.option('--length', default=25, help='Profile stack length')
@click.option('--profile-dir', default=None, help='Profile directory')
def profile(length, profile_dir):
    """Start the application under the code profiler."""
    # ...

@app.cli.command()
def deploy():
    """Run deployment tasks."""
    # ...
```

在本文中，我向您展示了新 Flask CLI 提供的功能,如果你想迁移基于 Flak-Script 的应用程序,这些功能将非常有用.还有几件事你可以做,但我没有在这里写到.文档描述了如何创建自己的驱动程序脚本，类似于 flask-script 的<code>manage.py</code>,以防你不想使用<code>flask</code>命令.它还告诉你如何通过项目的<code>setup.py</code>文件中定义的"entry points"来注册命令,如果你在项目中使用它的话.如果你感兴趣,请务必查看[CLI文档](http://flask.pocoo.org/docs/0.12/cli/)以了解这些内容.