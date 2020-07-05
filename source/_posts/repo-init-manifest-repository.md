---
title: Repo：init - 下载 Manifest 仓库
date: 2020-07-06 04:35:33
categories:
- 技术向
tags:
- Android
- Git
---

从前文知道，`init` 命令的执行是通过 `init.py` 的 `Execute` 方法完成实际操作。该方法主要功能为下载 Manifest 仓库，整体流程如下图所示：

![下载 Manifest 仓库](https://cdn.nlark.com/yuque/0/2020/svg/162088/1593033770887-eefdfc1d-40d9-4ab9-9a51-8e906d28d48c.svg#align=left&display=inline&height=830&margin=%5Bobject%20Object%5D&name=Repo-manifest%E4%BB%93%E5%BA%93.svg&originHeight=830&originWidth=491&size=15420&status=done&style=none&width=491)

<!--more-->

具体实现如下所示：

```python
  def Execute(self, opt, args):
    git_require(MIN_GIT_VERSION_HARD, fail=True)
......
    self._SyncManifest(opt)
    self._LinkManifest(opt.manifest_name)
......
    self._DisplayResult(opt)
```

`Execute` 主要通过调用 `_SyncManifest` 和 `_LinkManifest` 两个函数来完成 Manifest 仓库的克隆操作。

## 下载 Manifest 仓库

### _SyncManifest

`_SyncManifest` 的实现如下所示：

```python
  def _SyncManifest(self, opt):
    m = self.manifest.manifestProject
    is_new = not m.Exists

    if is_new:
......
      m._InitGitDir(mirror_git=mirrored_manifest_git)
......
    else:
      if opt.manifest_branch:
        m.revisionExpr = opt.manifest_branch
      else:
        m.PreSync()
......

    self._ConfigureDepth(opt)
......
    if not m.Sync_NetworkHalf(is_new=is_new, quiet=opt.quiet, verbose=opt.verbose,
                              clone_bundle=opt.clone_bundle,
                              current_branch_only=opt.current_branch_only,
                              tags=opt.tags, submodules=opt.submodules,
                              clone_filter=opt.clone_filter):
......

    if opt.manifest_branch:
      m.MetaBranchSwitch(submodules=opt.submodules)
......
    m.Sync_LocalHalf(syncbuf, submodules=opt.submodules)
......
    if is_new or m.CurrentBranch is None:
      if not m.StartBranch('default'):
        print('fatal: cannot create default in manifest', file=sys.stderr)
        sys.exit(1)
```

我们看到，大部分操作是通过一个 `m` 对象来完成操作的，从 `main.py` 文件的 `_Run` 方法看到，其中通过 `cmd.manifest = XmlManifest(cmd.repodir)` 给 `cmd` 对象注入了 `XmlManifest` 对象，而这里的 `m` 对象是 `XmlManifest` 对象中的 `MetaProject` 对象，XmlManifest 对象定义在 `.repo/repo/manifest_xml.py` 文件中。我们先放下当前流程去了解下这个对象。

### XmlManifest

查看类构造函数：

```python
class XmlManifest(object):
  """manages the repo configuration file"""

  def __init__(self, repodir):
    self.repodir = os.path.abspath(repodir)
    self.topdir = os.path.dirname(self.repodir)
    self.manifestFile = os.path.join(self.repodir, MANIFEST_FILE_NAME)
......
    self.repoProject = MetaProject(self, 'repo',
                                   gitdir=os.path.join(repodir, 'repo/.git'),
                                   worktree=os.path.join(repodir, 'repo'))
    self.manifestProject = MetaProject(self, 'manifests',
                     gitdir=os.path.join(repodir, 'manifests.git'),
                     worktree=os.path.join(repodir, 'manifests'))
......
```

它描述了项目的 Repo 目录（repodir）、根目录（topdir）和 Manifest.xml 文件（manifestFile），以及两个 MetaProject 对象描述了项目的 Repo 仓库（repoProject）和 Manifest 仓库（manifestProject）。

我们看到 Repo 仓库的位置如之前所述的，git 目录为 `repo/.git`，工作区为 `repo`，这与普通的 Git 仓库一样。而 Manifest 仓库的 Git 目录为 `manifests.git`，工作区为 `manifests`（为什么不一样？）。

在 Repo 项目中，每一个子项目（或者说子仓库）都用一个 Project 对象来描述。Project 类定义在文件 `.repo/repo/project.py` 文件中，用来封装对各个项目的基础 Git 操作，例如，对项目进行暂存、提交和更新等。Repo 仓库和 Manifest 仓库也是属于 Repo 项目中的仓库，但由于它们是用来描述 Repo 子项目元信息的仓库，我们就使用另外一个类型为 MetaProject 的对象来描述它们。不过实际的功能和 Project 对象是一致的，只是为了强调它们是元信息仓库而已。

### _SyncManifest（续）

回到 Init 类的成员函数 `_SyncManifest` 流程，我们可以看到，它整个流程主要调用了 Manifest 的 ManifestProject 对象的几个方法：

1. `m._InitGitDir`
1. `m.Sync_NetworkHalf`
1. `m.Sync_LocalHalf`
1. `m.StartBranch`

这几个函数实际上都是由 `MetaProject` 的父类 `Project` 来实现的，因此，下面我们就分析 Project 类的这几个函数的实现。

#### Project#_InitGitDir

`_InitGitDir` 用来初始化 Git 目录，具体实现如下所示：

```python
  def _InitGitDir(self, mirror_git=None, force_sync=False, quiet=False):
    init_git_dir = not os.path.exists(self.gitdir)
    init_obj_dir = not os.path.exists(self.objdir)
    try:
      # Initialize the bare repository, which contains all of the objects.
      if init_obj_dir:
        os.makedirs(self.objdir)
        self.bare_objdir.init()
......
```

首先是检查项目的 Git 目录是否已经存在。如果不存在，那么就会首先创建这个 Git 目录，然后再调用成员变量 `bare_git`（`_GitGetByExec` 对象）的成员函数 `init` 在该目录下初始化一个 Git 仓库。由于 `_GitGetByExec` 类不存在 `init` 成员函数，此时会触发调用 `__getattr__`。

```python
class Project(object):
  ......

  class _GitGetByExec(object):
    ......

    def __getattr__(self, name):
      """Allow arbitrary git commands using pythonic syntax.
      This allows you to do things like:
        git_obj.rev_parse('HEAD')
      Since we don't have a 'rev_parse' method defined, the __getattr__ will
      run.  We'll replace the '_' with a '-' and try to run a git command.
      Any other positional arguments will be passed to the git command, and the
      following keyword arguments are supported:
        config: An optional dict of git config options to be passed with '-c'.
      Args:
        name: The name of the git command to call.  Any '_' characters will
            be replaced with '-'.
      Returns:
        A callable object that will try to call git with the named command.
      """
......
      return runner
```

从注释可以知道，调用 `_GitGetByExec` 类没有实现的成员函数，会将调用的名称中的 `_` 替换为 `-` 并作为 `git` 的一个参数来执行。所以，当执行 `_GitGetByExec.init()` 的时候，实际上是执行了一个 `git init` 命令，它正是用来初始化 Git 仓库的命令。

#### Project#Sync_NetworkHalf

`Sync_NetworkHalf` 会通过网络来获取仓库内容，本地的工作区状态不会受到影响。其中命名中的 half 实际上是指这一步骤是 Sync 步骤的“一半”。具体实现如下：

```python
def Sync_NetworkHalf(self, quiet=False, verbose=False, is_new=None, current_branch_only=False, force_sync=False,
                     clone_bundle=True, tags=True, archive=False, optimized_fetch=False, retry_fetches=0,
                     prune=False, submodules=False, clone_filter=None):
......
    self._InitRemote()
......
    # See if we can skip the network fetch entirely.
    if not (optimized_fetch and
            (ID_RE.match(self.revisionExpr) and
             self._CheckForImmutableRevision())):
      if not self._RemoteFetch(
              initial=is_new, quiet=quiet, verbose=verbose, alt_dir=alt_dir,
              current_branch_only=current_branch_only,
              tags=tags, prune=prune, depth=depth,
              submodules=submodules, force_sync=force_sync,
              clone_filter=clone_filter, retry_fetches=retry_fetches):
        return False
......
    return True
```

主要操作为通过 `_InitRemote` 更新远程仓库信息，再调用 `_RemoteFetch` 来从远程仓库更新本地仓库。查看 `_RemoteFetch` 的实现，其核心操作就是调用 `git fetch` 命令从远程仓库更新本地仓库。

#### Project#Sync_LocalHalf

Sync 步骤的“另一半”是在工作区进行内容检出，具体实现如下：

```python
  def Sync_LocalHalf(self, syncbuf, force_sync=False, submodules=False):
    """Perform only the local IO portion of the sync process.
       Network access is not required.
    """
......
    self._InitWorkTree(force_sync=force_sync, submodules=submodules)
    all_refs = self.bare_ref.all
    self.CleanPublishedCache(all_refs)
    revid = self.GetRevisionId(all_refs)
......
    head = self.work_git.GetHead()
    if head.startswith(R_HEADS):
      branch = head[len(R_HEADS):]
      try:
        head = all_refs[head]
      except KeyError:
        head = None
    else:
      branch = None

    if branch is None or syncbuf.detach_head:
      # Currently on a detached HEAD.  The user is assumed to
      # not have any local modifications worth worrying about.
      #
......
      if head == revid:
        # No changes; don't do anything further.
        # Except if the head needs to be detached
        #
        if not syncbuf.detach_head:
          # The copy/linkfile config may have changed.
          self._CopyAndLinkFiles()
          return
      else:
        lost = self._revlist(not_rev(revid), HEAD)
        if lost:
          syncbuf.info(self, "discarding %d commits", len(lost))
      try:
        self._Checkout(revid, quiet=True)
......
      self._CopyAndLinkFiles()
      return
......
```

初始化新 Repo 项目时，流程较为简单，首先初始化工作区，再获取要 Checkout 的节点 ID，然后调用 `_Checkout` 方法检出仓库内容。查看 `_Checkout` 实现，就是通过 `git checkout` 命令完成实际操作。由于此时并没有分支信息，检出时是处于游离分支的状态。

#### Project#StartBranch

Menifest 仓库检出后，在初始化时会调用 `m.StartBranch('default')` 来创建一个 default 分支。至此，Repo 项目的 Manifest 仓库就完成了下载。

## 参考链接

> Repo 系列文章是基于老罗的[《Android源代码仓库及其管理工具Repo分析》](https://blog.csdn.net/Luoshengyang/article/details/18195205?utm_source=blogxgwz2)博客学习演绎的结果，原博文发布时间较早（ 2014-01-20），与现在有了些微差别并且有部分说法不太正确，因此重新学习整理出此系列文章，特此说明并表示感谢。

1. [Repo 项目主页](https://gerrit.googlesource.com/git-repo/)
1. [Android 源代码仓库及其管理工具 Repo 分析](https://blog.csdn.net/Luoshengyang/article/details/18195205?utm_source=blogxgwz2)

### 附：Project 对象

Project 对象构造函数如下所示：

```python
class Project(object):
......
  def __init__(self, manifest, name, remote, gitdir, objdir, worktree, relpath,
               revisionExpr, revisionId,
               rebase=True, groups=None,
               sync_c=False, sync_s=False, sync_tags=True, clone_depth=None,
               upstream=None, parent=None, use_git_worktrees=False,
               is_derived=False, dest_branch=None, optimized_fetch=False,
               retry_fetches=0, old_revision=None):
    """Init a Project object.

    Args:
      manifest: The XmlManifest object.
      name: The `name` attribute of manifest.xml's project element.
      remote: RemoteSpec object specifying its remote's properties.
      gitdir: Absolute path of git directory.
      objdir: Absolute path of directory to store git objects.
      worktree: Absolute path of git working tree.
      relpath: Relative path of git working tree to repo's top directory.
      revisionExpr: The `revision` attribute of manifest.xml's project element.
      revisionId: git commit id for checking out.
      rebase: The `rebase` attribute of manifest.xml's project element.
      groups: The `groups` attribute of manifest.xml's project element.
      sync_c: The `sync-c` attribute of manifest.xml's project element.
      sync_s: The `sync-s` attribute of manifest.xml's project element.
      sync_tags: The `sync-tags` attribute of manifest.xml's project element.
      upstream: The `upstream` attribute of manifest.xml's project element.
      parent: The parent Project object.
      use_git_worktrees: Whether to use `git worktree` for this project.
      is_derived: False if the project was explicitly defined in the manifest;
                  True if the project is a discovered submodule.
      dest_branch: The branch to which to push changes for review by default.
      optimized_fetch: If True, when a project is set to a sha1 revision, only
                       fetch from the remote if the sha1 is not present locally.
      retry_fetches: Retry remote fetches n times upon receiving transient error
                     with exponential backoff and jitter.
      old_revision: saved git commit id for open GITC projects.
    """
......
```

- manifest：XmlManifest 对象。
- name：项目名称，也是 Git 远程仓库中的名称。通过它和远程链接拼接出实际的远程仓库链接：`${remote_fetch}/${project_name}.git`
- remote：远程仓库。
- gitdir：.git 目录，也就是 Git 仓库目录。
- objdir：存放 git 对象的目录，通常与 gitdir 等同。
- worktree：项目的工作区，即仓库检出内容的目录。
- relpath：项目的工作区相对 Repo 项目的根目录的相对路径。
- revisionExpr：项目要跟踪的 Git  分支的名称。
- revisionId：项目跟踪的 Git 提交 ID。
- rebase：_**Manifest 格式中无此字段**_
- groups：项目所属的分组列表。
- sync_c：为 true 时表示 sync 时仅同步给定的分支。
- sync_s：为 true 时表示 sync 时同时同步子项目。
- sync_tags：为 true 时表示 sync 时会同步 Tag。
- upstream：项目的远程分支引用。
- parent：项目的父项目。
- use_git_worktrees：是否使用 git worktree。
- is_derived：当前这个项目是否是项目的子项目。
- dest_branch：推送后的远程 Review 的分支。
- optimized_fetch、retry_fetches、old_revision：这几个暂时不了解。
