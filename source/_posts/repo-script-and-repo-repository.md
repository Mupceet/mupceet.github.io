---
title: Repo：Repo 脚本与 Repo 仓库
date: 2020-05-24 09:57:33
categories:
- 技术向
tags:
- Android
- Git
---

Google 选择了 Git 作为 AOSP 的版本控制系统，而 AOSP 一方面为了方便项目的独立开发，另一方面也为了方便对部分组件进行定制，拆分成数百个 Git 项目。为了更好地管理这些仓库，创建了 Repo 工具进行批量的 Git 仓库管理。Repo 是基于 Git 构建的工具，因此，它的工作流与普通的 Git 工作流类似，它的存在是为了让 Git 在如此大的项目系统中更加好用。

## 介绍

一个完整的使用 repo 管理的项目由三部分组成：

1. 子项目仓库
1. Manifest 仓库
1. Repo 仓库

以 AOSP 为例，根据功能和模块，项目包括了数百个 Git 仓库，比如 framework 目录下有多个 Git 仓库，这些仓库就是众多的**子项目仓库**。而不同的 AOSP 版本、不同的定制产物版本，包含的子项目不完全相同，因此，通过一个 Git 仓库来管理各个版本的子项目信息，这个仓库就称为 **Manifest 仓库**。Repo 工具通过 Manifest 仓库里的信息对整个项目中的子项目进行操作，而 Repo 工具实际上是由一系列的 Python 脚本组成的，这些 Python 脚本通过调用 Git 命令来完成功能，这些脚本本身也通过 Git 仓库来管理，这个仓库就称为 **Repo 仓库**，并且我们每次执行 Repo 命令的时候，Repo 仓库都会对自己进行一次更新。

<!--more-->

现在，我们用一个图来勾勒一下它们的关系：

![关系图](https://github.com/Mupceet/article-piture/raw/master/repo-script-and-projects.png)

根据 AOSP 项目主页介绍，项目要从一个 Repo 脚本开始。我们根据指导，下载该 Repo 脚本，赋予可执行权限，并将它添加到环境变量中，以便执行后续的 repo 命令。

```shell
$ mkdir -p ~/.bin
$ PATH="${HOME}/.bin:${PATH}"
$ curl https://storage.googleapis.com/git-repo-downloads/repo > ~/.bin/repo
$ chmod a+rx ~/.bin/repo
```

以下我们以 Jatpack Compose 项目为例进行说明。下载好 Repo 脚本之后，可以通过以下命令来下载一个 Repo 项目：

```shell
$ repo init -u https://android.googlesource.com/platform/manifest -b androidx-master-dev
```

事实上，这个命令包含了两部分操作：下载 Repo 仓库和下载 Manifest 仓库。而若是非要拆开执行的话，可以执行 `repo init` 命令仅下载 Repo 仓库，不过这样的话执行后会有错误提示。

```shell
$ repo init
```

## 下载 Repo 仓库

我们先看下 Repo 仓库是如何下载的。整体流程如下图所示：

![Repo 仓库初始化](https://raw.githubusercontent.com/Mupceet/article-piture/master/repo-init-base.png)

```python
def main(orig_args):
  cmd, opt, args = _ParseArguments(orig_args)

  repo_main, rel_repo_dir = None, None
...
    repo_main, rel_repo_dir = _FindRepo()

  wrapper_path = os.path.abspath(__file__)
  my_main, my_git = _RunSelf(wrapper_path)

  cwd = os.getcwd()
...
  if not repo_main:
    if opt.help:
      _Usage()
    if cmd == 'help':
      _Help(args)
    if not cmd:
      _NotInstalled()
    if cmd == 'init' or cmd == 'gitc-init':
      if my_git:
        _SetDefaultsTo(my_git)
      try:
        _Init(args, gitc_init=(cmd == 'gitc-init'))
      except CloneFailure:
...
        sys.exit(1)
      repo_main, rel_repo_dir = _FindRepo()
    else:
      _NoCommands(cmd)

  if my_main:
    repo_main = my_main

  ver_str = '.'.join(map(str, VERSION))
  me = [sys.executable, repo_main,
        '--repo-dir=%s' % rel_repo_dir,
        '--wrapper-version=%s' % ver_str,
        '--wrapper-path=%s' % wrapper_path,
        '--']
  me.extend(orig_args)
  exec_command(me)
  print("fatal: unable to start %s" % repo_main, file=sys.stderr)
  sys.exit(148)

if __name__ == '__main__':
  main(sys.argv[1:])
```

### _ParseArguments

`_ParseArguments` 对 Repo 的参数进行解析，得到要执行的命令及其对应的参数。其中第一个非 `-` 开头的参数作为命令，后续部分作为命令的参数。

例如，调用 `repo init -u URL` 的时候，就表示要执行的命令是 `init`，这个命令后面跟的参数是 `-u URL`。如果调用 `repo sync`，就表示要执行的命令是 `sync`，后续没有参数。

`_ParseArguments` 的实现如下所示：

```python
def _ParseArguments(args):
  cmd = None
  opt = _Options()
  arg = []

  for i in range(len(args)):
    a = args[i]
    if a == '-h' or a == '--help':
      opt.help = True

    elif not a.startswith('-'):
      cmd = a
      arg = args[i + 1:]
      break
  return cmd, opt, arg
```

### _FindRepo

`_FindRepo` 从执行指令的当前目录开始往上遍历直到根目录。如果中间某一个目录存在一个 `.repo/repo/main.py` 文件，那么就代表找到 Repo 仓库了，这时候它就会返回 main.py 文件的绝对路径和 .repo 目录的绝对路径。

`_FindRepo` 的实现如下所示：

```python
repodir = '.repo'               # name of repo's private directory
S_repo = 'repo'                 # special repo repository
REPO_MAIN = S_repo + '/main.py' # main script

def _FindRepo():
  """Look for a repo installation, starting at the current directory.
  """
  curdir = os.getcwd()
  repo = None

  olddir = None
  while curdir != '/' \
          and curdir != olddir \
          and not repo:
    repo = os.path.join(curdir, repodir, REPO_MAIN)
    if not os.path.isfile(repo):
      repo = None
      olddir = curdir
      curdir = os.path.dirname(curdir)
  return (repo, os.path.join(curdir, repodir))
```

### _RunSelf

`_RunSelf` 检查 Repo 脚本所在目录是否存在一个 Repo 仓库：

```python
def _RunSelf(wrapper_path):
  my_dir = os.path.dirname(wrapper_path)
  my_main = os.path.join(my_dir, 'main.py')
  my_git = os.path.join(my_dir, '.git')

  if os.path.isfile(my_main) and os.path.isdir(my_git):
    for name in ['git_config.py',
                 'project.py',
                 'subcmds']:
      if not os.path.exists(os.path.join(my_dir, name)):
        return None, None
    return my_main, my_git
  return None, None
```

从这里我们就可以看出，一个标准的 Repo 仓库应用有：

1. 存在一个 main.py 文件；
1. 存在一个 .git 目录；
1. 存在一个 git_config.py 文件；
1. 存在一个 project.py 文件；
1. 存在一个 subcmds 目录。

再回到 `main` 函数中。如果调用 `_FindRepo` 得到的 `repo_main` 的值等于空，那么就说明当前目录还没有安装 Repo 仓库，这时候 repo 后面所跟的参数只能是 `help` 或者 `init`，否则就会显示错误信息。如果 repo 后面跟的参数是 `help`，就打印出 Repo 脚本的帮助文档。

我们关注 repo 后面跟的参数是 `init` 的情况。这时候看一下调用 `_RunSelf` 的返回值 `my_git` 是否不等于空。如果不等于空的话，那么就说明 Repo 脚本所在目录存一个 Repo 仓库，这时候就调用 `_SetDefaultsTo` 更新要克隆的 Repo 仓库的源和分支。如果等于空，就将 `https://gerrit.googlesource.com/git-repo` 作为默认的 Repo 仓库源，`stable` 作为要克隆的分支。

### _SetDefaultsTo

`_SetDefaultsTo` 的实现如下所示：

```python
GIT = 'git'
REPO_URL = os.environ.get('REPO_URL', None)
if not REPO_URL:
  REPO_URL = 'https://gerrit.googlesource.com/git-repo'
REPO_REV = 'stable'

def _SetDefaultsTo(gitdir):
  global REPO_URL
  global REPO_REV

  REPO_URL = gitdir
  proc = subprocess.Popen([GIT,
                           '--git-dir=%s' % gitdir,
                           'symbolic-ref',
                           'HEAD'],
                          stdout = subprocess.PIPE,
                          stderr = subprocess.PIPE)
  REPO_REV = proc.stdout.read().strip()
  proc.stdout.close()

  proc.stderr.read()
  proc.stderr.close()

  if proc.wait() != 0:
    _print('fatal: %s has no current branch' % gitdir, file=sys.stderr)
    sys.exit(1)
```

> Git 知识点：git --git-dir=[.git dir path] symbolic-ref HEAD 获取指定的 git 仓库路径 HEAD 指向的分支名

到目前为止，我们可以知道，在执行 `repo init` 命令，有以下两种情况：

1. 如果我们只是从网上下载了一个 Repo 脚本，那么就会从**远程**克隆一个 Repo 仓库到当前执行命令的目录中来。
1. 如果我们从网上下载的是一个带有 Repo 仓库的 Repo 脚本，那么就会从**本地**克隆一个 Repo 仓库到当前执行命令的目录中来。

### _Init

我们再继续看 main 函数的实现，它接下来调用 `_Init` 在当前执行 Repo 脚本的目录下创建一个 Repo 仓库：

```python
def _Init(args, gitc_init=False):
  """Installs repo by cloning it over the network.
  """
......
  opt, args = init_optparse.parse_args(args)
  if args:
    init_optparse.print_usage()
    sys.exit(1)

  url = opt.repo_url
  if not url:
    url = REPO_URL
    extra_args.append('--repo-url=%s' % url)

  branch = opt.repo_branch
  if not branch:
    branch = REPO_REV
    extra_args.append('--repo-branch=%s' % branch)

  if branch.startswith('refs/heads/'):
    branch = branch[len('refs/heads/'):]
......
  
  try:
......
    os.mkdir(repodir)
  except OSError as e:
......
  _CheckGitVersion()
  try:
    if NeedSetupGnuPG():
      can_verify = SetupGnuPG(opt.quiet)
    else:
      can_verify = True

    dst = os.path.abspath(os.path.join(repodir, S_repo))
    _Clone(url, dst, opt.quiet, not opt.no_clone_bundle)

    if can_verify and not opt.no_repo_verify:
      rev = _Verify(dst, branch, opt.quiet)
    else:
      rev = 'refs/remotes/origin/%s^0' % branch

    _Checkout(dst, branch, rev, opt.quiet)

    if not os.path.isfile(os.path.join(dst, 'repo')):
      print("warning: '%s' does not look like a git-repo repository, is "
            "REPO_URL set correctly?" % url, file=sys.stderr)

  except CloneFailure:
......
    raise
```

首先确认 Repo 仓库的源地址和分支。如果没有通过 `--repo-url` 和 `--repo-branch` 来指定 Repo 仓库的源地址和分支，那么就使用由 `REPO_URL` 和 `REPO_REV` 所指定的源地址和分支。从前面的分析可以知道，`REPO_URL` 和 `REPO_REV` 要么指向远程地址和 stable 分支，要么指向 Repo 脚本所在目录的 Repo 仓库和该仓库的当前分支。

然后创建 .repo 目录。并且检查系统的 Git 版本，检查是否需要验证等一会克隆回来的 Repo 仓库的 GPG。如果需要验证的话，那么就会在调用 `_Clone` 函数完成克隆 Repo 仓库之后，调用 `_Verify` 函数来验证该 Repo 仓库的 GPG。

最关键的地方就在于 `_Clone` 函数和 `_Checkout` 函数的调用，前者把 Repo 仓库克隆到当前目录下的 `.repo/repo` 目录中，后者在克隆回来的仓库中将对应分支 checkout 出来。

### _Clone

`_Clone` 的实现如下所示：

```python
def _Clone(url, local, quiet, clone_bundle):
  """Clones a git repository to a new subdirectory of repodir
  """
  try:
    os.mkdir(local)
  except OSError as e:
......
    raise CloneFailure()

  cmd = [GIT, 'init', '--quiet']
  try:
    proc = subprocess.Popen(cmd, cwd=local)
  except OSError as e:
......

  _InitHttp()
  _SetConfig(local, 'remote.origin.url', url)
  _SetConfig(local,
             'remote.origin.fetch',
             '+refs/heads/*:refs/remotes/origin/*')
  if clone_bundle and _DownloadBundle(url, local, quiet):
    _ImportBundle(local)
  _Fetch(url, local, 'origin', quiet)
```

> Git 知识点
>
> 1. git init --quiet 静默地初始化一个 Git 仓库，仅打印错误信息
> 1. git config name vale 设置 git 相关配置

这个函数首先是调用 `git init` 在当前目录下的 `.repo/repo` 子目录初始化为 Git 仓库。接着读取环境变量中的 `http_proxy` 来设置代理。然后调用 `_SetConfig` 函来设置该 Git 仓库的远程信息。
再接着调用 `_DownloadBundle` 来从远程下载 `clone.bundle` 文件。

### _DownloadBundle

Bundle 文件是 Git 提供的一种机制，使用 git bundle 命令为 Git 仓库创建一个 Bundle 文件，这个 Bundle 文件就会包含 Git 仓库的信息。把这个 Bundle 文件通过其它方式拷贝到另一台机器上，就可以将它作为一个本地 Git 仓库来使用，而不用去访问远程网络。

`_DownloadBundle` 的实现如下所示：

```python
def _DownloadBundle(url, local, quiet):
  if not url.endswith('/'):
    url += '/'
  url += 'clone.bundle'

  proc = subprocess.Popen(
      [GIT, 'config', '--get-regexp', 'url.*.insteadof'],
      cwd=local,
      stdout=subprocess.PIPE)
  for line in proc.stdout:
    line = line.decode('utf-8')
    m = re.compile(r'^url\.(.*)\.insteadof (.*)$').match(line)
    if m:
      new_url = m.group(1)
      old_url = m.group(2)
      if url.startswith(old_url):
        url = new_url + url[len(old_url):]
        break
  proc.stdout.close()
  proc.wait()

  if not url.startswith('http:') and not url.startswith('https:'):
    return False

  dest = open(os.path.join(local, '.git', 'clone.bundle'), 'w+b')
  try:
    try:
      r = urllib.request.urlopen(url)
......
    try:
......
      while True:
        buf = r.read(8192)
        if not buf:
          return True
        dest.write(buf)
    finally:
      r.close()
  finally:
    dest.close()
```

> Git 知识点：
>
> 1. git config --get-regexp key 返回正则匹配的配置值
> 1. `url."https://".insteadOf "git://"` 可以统一配置传输协议

首先，根据前面的 URL 信息以及配置，获取最终的仓库远程链接。如果是本地仓库，这里直接返回 False，否则就从网络上下载这个 bundle 文件。

如果成功下载到该 clone.bundle 文件，接下来就调用函数 `_ImportBundle` 将它作为源仓库克隆为新的 Repo 仓库。

`_ImportBundle` 的实现如下所示：

```python
def _ImportBundle(local):
  path = os.path.join(local, '.git', 'clone.bundle')
  try:
    _Fetch(local, local, path, True)
  finally:
    os.remove(path)
```

结合 `_Clone` 函数和 `_ImportBundle` 函数就可以看出，从远程 url 下载到 clone.bundle 文件后再创建克隆 Repo 仓库和从脚本同目录的本地克隆 Repo 仓库都是通过 `_Fetch` 来实现的。区别就在于调用函数 `_Fetch` 时指定的第三个参数，前者是下载到本地的 clone.bundle 文件路径，后者是 origin（表示本地的远程仓库名称）。

### _Fetch

`_Fetch` 的实现如下所示：

```python
def _Fetch(url, local, src, quiet):
  if not quiet:
    print('Get %s' % url, file=sys.stderr)

  cmd = [GIT, 'fetch']
  if quiet:
    cmd.append('--quiet')
    err = subprocess.PIPE
  else:
    err = None
  cmd.append(src)
  cmd.append('+refs/heads/*:refs/remotes/origin/*')
  cmd.append('+refs/tags/*:refs/tags/*')

  proc = subprocess.Popen(cmd, cwd=local, stderr=err)
  if err:
    proc.stderr.read()
    proc.stderr.close()
  if proc.wait() != 0:
    raise CloneFailure()
```

> Git 知识点：git fetch src \<refspec> 指定拉取的分支、Tag

函数 `_Fetch` 实际上就是通过 `git fetch` 从指定的仓库源克隆一个新的 Repo 仓库到当前目录下的 `.repo/repo` 子目录中。

到这里，已经完成 Repo 仓库的获取，接下来还需要从这个 Repo 仓库 checkout 出一个分支来，才能正常工作。从 Repo 仓库 checkout 出一个分支是通过调用函数 `_Checkout` 来实现的：

### _Checkout

`_Checkout` 的实现如下所示：

```python
def _Checkout(cwd, branch, rev, quiet):
  """Checkout an upstream branch into the repository and track it.
  """
  cmd = [GIT, 'update-ref', 'refs/heads/default', rev]
  if subprocess.Popen(cmd, cwd=cwd).wait() != 0:
    raise CloneFailure()

  _SetConfig(cwd, 'branch.default.remote', 'origin')
  _SetConfig(cwd, 'branch.default.merge', 'refs/heads/%s' % branch)

  cmd = [GIT, 'symbolic-ref', 'HEAD', 'refs/heads/default']
  if subprocess.Popen(cmd, cwd=cwd).wait() != 0:
    raise CloneFailure()

  cmd = [GIT, 'read-tree', '--reset', '-u']
  if not quiet:
    cmd.append('-v')
  cmd.append('HEAD')
  if subprocess.Popen(cmd, cwd=cwd).wait() != 0:
    raise CloneFailure()
```

> Git 知识点：
>
> 1. git update-ref \<ref> \<newvalue> 将新目标存储到引用上
> 1. git symbolic-ref \<ref> \<ref> 创建或更新引用指向的对象
> 1. git read-tree HEAD 将信息读到引用上

要 checkout 出来的分支由参数 `branch` 指定。从前面的分析可以知道，如果当前执行的 Repo 脚本所在目录存在一个 Repo 仓库，那么参数 `branch` 描述的就是该仓库当前 checkout 出来的分支。否则的话，参数 `branch` 描述的就是从远程仓库克隆回来的 stable 分支。

到这里，就完成了 Repo 仓库的下载。

## Repo 命令执行

回到 main 函数：

```python
def main(orig_args):
......
  ver_str = '.'.join(map(str, VERSION))
  me = [sys.executable, repo_main,
        '--repo-dir=%s' % rel_repo_dir,
        '--wrapper-version=%s' % ver_str,
        '--wrapper-path=%s' % wrapper_path,
        '--']
  me.extend(orig_args)
  exec_command(me)
  print("fatal: unable to start %s" % repo_main, file=sys.stderr)
  sys.exit(148)
......
```

在完成 Repo 仓库下载或找到 Repo 仓库后，可以看到实际上是通过 Repo 仓库中的 main.py 脚本执行对应的命令。其中，将命令拼接成如下格式调用执行:

```shell
['/usr/bin/python3.7', '[path]/main.py', '--repo-dir=[path]/.repo', '--wrapper-version=2.5', '--wrapper-path=[path]/repo', '--', 'init', '-u', 'https://android.googlesource.com/platform/manifest', '-b', 'androidx-master-dev']
```

main.py 脚本的 `_Main` 函数如下所示：

```python
def _Main(argv):
  result = 0
......
  repo = _Repo(opt.repodir)
  try:
    try:
......
      name, gopts, argv = repo._ParseArgs(argv)
      run = lambda: repo._Run(name, gopts, argv) or 0
......
        result = run()
    finally:
      close_ssh()
  except KeyboardInterrupt:
......
  sys.exit(result)
```

### _Run

从 `main` 方法的返回结果可以看出，实际的命令执行是 `repo._Run` 方法，我们先跳过 `_Repo` 类，直接查看它的 `_Run` 方法。

`_Run` 的实现如下所示：

```python
  def _Run(self, name, gopts, argv):
    """Execute the requested subcommand."""
    result = 0
......
    try:
      cmd = self.commands[name]()
    except KeyError:
......
      return 1
    cmd.repodir = self.repodir
    cmd.manifest = XmlManifest(cmd.repodir)
    cmd.gitc_manifest = None
......
    Editor.globalConfig = cmd.manifest.globalConfig
......
    try:
      copts, cargs = cmd.OptionParser.parse_args(argv)
      copts = cmd.ReadEnvironmentOptions(copts)
    except NoManifestException as e:
......
      return 1
......
    start = time.time()
    cmd_event = cmd.event_log.Add(name, event_log.TASK_COMMAND, start)
    cmd.event_log.SetParent(cmd_event)
    try:
      cmd.ValidateOptions(copts, cargs)
      result = cmd.Execute(copts, cargs)
    except (DownloadError, ManifestInvalidRevisionError,
            NoManifestException) as e:
......
    return result
```

由 `result = cmd.Execute(copts, cargs)` 可以知道，命令实际是由 `cmd` 对象来执行，通过给 `cmd` 注入对应的参数，再调用它的 `Execute` 方法完成操作。而 `cmd` 对象是从一个 `all_commands` 表中根据命令的 `name` 获取的，那这个命令表是如何构建起来的呢？原来在 Repo 仓库的 subcmds 目录中，有一个`__init__.py` 文件，当 subcmds 被 import 时，定义在它里面的命令就会被执行，如下所示：

```python
all_commands = {}

my_dir = os.path.dirname(__file__)
for py in os.listdir(my_dir):
  if py == '__init__.py':
    continue

  if py.endswith('.py'):
    name = py[:-3]

    clsn = name.capitalize()
    while clsn.find('_') > 0:
      h = clsn.index('_')
      clsn = clsn[0:h] + clsn[h + 1:].capitalize()

    mod = __import__(__name__,
                     globals(),
                     locals(),
                     ['%s' % name])
    mod = getattr(mod, name)
    try:
      cmd = getattr(mod, clsn)
    except AttributeError:
      raise SyntaxError('%s/%s does not define class %s' % (
          __name__, py, clsn))

    name = name.replace('_', '-')
    cmd.NAME = name
    all_commands[name] = cmd

# Add 'branch' as an alias for 'branches'.
all_commands['branch'] = all_commands['branches']
```

`__init__.py` 会列出 subcmds 目录中的所有 Python 文件，并且里面找到对应的类，然后再创建这个类的一个对象，并且以文件名为关键字将该对象保存在全局变量 `all_commands` 中（除了`__init__.py`）。例如，对于 `init.py` 文件，它的文件名称去掉后缀名后为 `init`，再将 init 的首字母大写，得到 `Init`。也就是说，init.py 需要定义一个 `Init` 类，并且这个类需要直接或者间接地从 `Command` 类继承下来。而 `Command` 类有一个成员函数 `Execute`，它的各个子类需要对它进行重写，以实现各自的功能。这样，在 main.py 要执行相应命令时，就可以调用它对应的 `Command` 对象的 `Execute` 方法完成实际的命令操作。

## 参考链接

> Repo 系列文章是基于老罗的[《Android源代码仓库及其管理工具Repo分析》](https://blog.csdn.net/Luoshengyang/article/details/18195205?utm_source=blogxgwz2)博客学习演绎的结果，原博文发布时间较早（ 2014-01-20），与现在有了些微差别并且有部分说法不太正确，因此重新学习整理出此系列文章，特此说明并表示感谢。

1. [Repo 项目主页](https://gerrit.googlesource.com/git-repo/)
1. [Android 源代码仓库及其管理工具 Repo 分析](https://blog.csdn.net/Luoshengyang/article/details/18195205?utm_source=blogxgwz2)
