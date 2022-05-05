# 使用 OBS 在本地构建 openEuler 的软件包

## OBS 简介

**O**pen **B**uild **S**ervice 是一套系统化的软件自动构建打包程序，也是 openSUSE 官方钦定的构建系统。

openEuler 系统也使用 OBS 来自动构建软件包。

第一次使用 OBS 的话可能会感觉有点绕，OBS 的逻辑是：项目 - 子项目 - 具体的包。

以 openEuler RISC-V 上的 gcc 举例：

[openEuler:Mainline](https://build.openeuler.org/project/show/openEuler:Mainline) 是主项目，这个主项目有 [openEuler:Mainline:RISC-V](https://build.openeuler.org/project/show/openEuler:Mainline:RISC-V) 子项目，在子项目下的 [gcc](https://build.openeuler.org/package/show/openEuler:Mainline:RISC-V/gcc) 则是我们要的 gcc。

## _service 文件

openEuler 的 OBS 仓库主要使用 xml 格式的 `_service`  文件进行文件管理。

最直接的，你可以把源代码，spec 配置文件等都上传到 OBS 仓库中，OBS 系统会自动根据你的 spec 文件进行编译，打包。

`_service` 则可以指定源代码仓库的位置，由 OBS 自动 clone 代码下来进行编译打包。

以 [htop](https://build.openeuler.org/package/show/openEuler:Mainline:RISC-V/htop) 举例，`_service`  文件内容如下：

```xml
<services>
    <service name="tar_scm">
      <param name="scm">git</param>
      <param name="url">git@gitee.com:src-openeuler/htop.git</param>
      <param name="exclude">.git</param>
      <param name="revision">624dd5dc09264bba2bfece2ab8d6a6a555a0c412</param>
    </service>
    <service name="extract_file">
        <param name="archive">*.tar</param>
        <param name="files">*/*</param>
    </service>
</services>
```

`tar_scm` 定义了源代码拉取方式，指从源代码仓库拉取对应的文件存档。

`scm` 对应了代码管理工具，`url` 和 `exclude` 就是字面含义。通过 ` revision` 字段来指定源代码的版本，可以指定具体的 hash、tag 或者 branch。

此部分内容的参考与拓展阅读：

https://en.opensuse.org/openSUSE:Build_Service_Concept_SourceService

https://openbuildservice.org/help/manuals/obs-user-guide/cha.obs.source_service.html

## OBS 前期配置

先安装 osc，同时安装 sudo，很简单 `dnf install osc sudo` 。

接着修改 osc 配置文件 `~/.config/osc/oscrc`

```ini
[general]
apiurl = https://build.openeuler.org/
no_verify = 1
 
[https://build.openeuler.org]
user = username
pass = password
```

## 软件包的拉取与构建

假设我们想在本地环境中尝试构建 htop 这个包，先从 openEuler:Mainline:RISC-V 仓库中拉取 htop。

```bash
~ osc checkout openEuler:Mainline:RISC-V htop
A    openEuler:Mainline:RISC-V/htop
A    openEuler:Mainline:RISC-V/htop/_service
At revision 4.
```

进入软件包所在目录 `openEuler:Mainline:RISC-V/htop`

此时仅有前文提到的 `_service` 文件

```bash
~ ll
total 4.0K
-rw-r--r-- 1 root root 423 Mar  1 14:18 _service
```

还需要从上游拉取我们需要的源代码：

```bash
~ osc update -S
A    _service:extract_file:htop-3.1.2.tar.gz
A    _service:extract_file:htop.spec
A    _service:extract_file:htop.yaml
A    _service:tar_scm:htop-1643246480.624dd5d.tar
At revision fc1300f4864270c97d05189eb772b29a.
```

此时的文件还比较杂乱，包含 _service 带来的文件前缀，osc 并不能正确识别我们需要的文件。

```bash
~ ll
total 784K
-rw-r--r-- 1 root root  423 Mar  1 14:18 _service
-rw-r--r-- 1 root root 379K Jan 27 09:21 _service:extract_file:htop-3.1.2.tar.gz
-rw-r--r-- 1 root root 1.4K Jan 27 09:21 _service:extract_file:htop.spec
-rw-r--r-- 1 root root   73 Nov  2 19:48 _service:extract_file:htop.yaml
-rw-r--r-- 1 root root 390K Feb 16 18:23 _service:tar_scm:htop-1643246480.624dd5d.tar
```

运行一个从官方那里偷来的 shell：

```shell
rm -f _service;for file in `ls`;do new_file=${file##*:};mv $file $new_file;done
```

```bash
~ ll
total 780K
-rw-r--r-- 1 root root 390K Feb 16 18:23 htop-1643246480.624dd5d.tar
-rw-r--r-- 1 root root 379K Jan 27 09:21 htop-3.1.2.tar.gz
-rw-r--r-- 1 root root 1.4K Jan 27 09:21 htop.spec
-rw-r--r-- 1 root root   73 Nov  2 19:48 htop.yaml
```

文件名已经比较正常了，可以开始构建了。

在这里以构建 [standard_riscv64](https://build.openeuler.org/package/binaries/openEuler:Mainline:RISC-V/htop/standard_riscv64) 为例：

```bash
~ sudo osc build standard_riscv64
Building htop.spec for standard_riscv64/riscv64
Getting buildinfo from server and store to /home/work/openEuler:Mainline:RISC-V/htop/.osc/_buildinfo-standard_riscv64-riscv64.xml
Getting buildconfig from server and store to /home/work/openEuler:Mainline:RISC-V/htop/.osc/_buildconfig-standard_riscv64-riscv64
Updating cache of required packages
...
```

然后等一会儿风扇起飞：

```bash
[  249s] ... checking for files with abuild user/group
[  249s] 
[  249s] openEuler-RISCV-rare finished "build htop.spec" at Thu Apr 28 20:44:43 UTC 2022.
[  249s] 

/var/tmp/build-root/standard_riscv64-riscv64/home/abuild/rpmbuild/SRPMS/htop-3.1.2-1.oe1.src.rpm
/var/tmp/build-root/standard_riscv64-riscv64/home/abuild/rpmbuild/RPMS/riscv64/htop-debuginfo-3.1.2-1.oe1.riscv64.rpm
/var/tmp/build-root/standard_riscv64-riscv64/home/abuild/rpmbuild/RPMS/riscv64/htop-debugsource-3.1.2-1.oe1.riscv64.rpm
/var/tmp/build-root/standard_riscv64-riscv64/home/abuild/rpmbuild/RPMS/riscv64/htop-3.1.2-1.oe1.riscv64.rpm
```

软件包已经按照我们的预期构建完成。

当然也可以使用 OBS 的 worker 在云端构建，可以参考[这篇文章](https://wiki.251.sh/openeuler_risc-v_obs)。

此部分内容的参考与拓展阅读：

https://en.opensuse.org/openSUSE:OSC

https://en.opensuse.org/images/d/df/Obs-cheat-sheet.pdf

https://wiki.251.sh/openeuler_risc-v_obs