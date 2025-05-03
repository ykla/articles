# 在台机上实现 UFS 日志功能

- 原文：[Implementing UFS Journaling on a Desktop PC](https://docs.freebsd.org/en/articles/gjournal-desktop/)

## 摘要

日志文件系统使用日志记录文件系统中发生的所有事务，并在系统崩溃或断电时保持其完整性。尽管仍然可能丢失未保存的文件更改，日志几乎完全消除了因非正常关机而导致的文件系统损坏的可能性。它还将故障后的文件系统检查所需的时间缩短到最小。尽管 FreeBSD 使用的 UFS 文件系统本身并未实现日志记录，但 FreeBSD 7.*X* 中的 GEOM 框架的新日志类可以提供与文件系统无关的日志功能。本文将解释如何在典型的桌面 PC 场景中实现 UFS 日志记录。

## 1. 介绍

虽然专业服务器通常能很好地防范突发的关机情况，但典型的桌面 PC 往往容易受到断电、意外重启和其他用户相关事件的影响，导致非正常关机。软更新通常能够有效地保护文件系统，尽管在大多数情况下仍然需要进行长时间的后台检查。在少数情况下，文件系统损坏可能严重到需要用户干预，并且可能导致数据丢失。

GEOM 提供的新日志功能在这种情况下可以大大帮助，通过几乎消除文件系统检查所需的时间，并确保文件系统迅速恢复到一致的状态。

本文描述了在典型桌面 PC 场景下实现 UFS 日志记录的过程（一个硬盘同时用于操作系统和数据）。它应该在 FreeBSD 的全新安装过程中进行。在安装过程中，步骤足够简单，不需要过于复杂的命令行交互。

阅读本文后，您将了解：

* 如何在新安装的 FreeBSD 中为日志记录预留空间。
* 如何加载并启用 `geom_journal` 模块（或在自定义内核中构建对其的支持）。
* 如何将现有文件系统转换为使用日志记录，并在 **/etc/fstab** 中使用哪些选项进行挂载。
* 如何在新（空）分区中实现日志记录。
* 如何排查与日志记录相关的常见问题。

在阅读本文之前，您应该能够：

* 理解基本的 UNIX® 和 FreeBSD 概念。
* 熟悉 FreeBSD 的安装过程和 sysinstall 工具。

>**警告**
>
>本文所述的过程旨在为全新安装做准备，在此过程中磁盘上尚未存储任何实际用户数据。尽管可以修改和扩展此过程以适应已投入生产的系统，但在进行此操作之前，您应当 *备份* 所有重要数据。低级别操作磁盘和分区可能导致致命错误和数据丢失。


## 2. 在 FreeBSD 中理解日志记录

FreeBSD 7.*X* 中由 GEOM 提供的日志记录功能并非特定于某个文件系统（例如 Linux® 中的 ext3 文件系统），而是作用于块级别。尽管这意味着它可以应用于不同的文件系统，但对于 FreeBSD 7.0-RELEASE，它仅能用于 UFS2。

此功能通过将 **geom\_journal.ko** 模块加载到内核中（或将其构建到自定义内核中）并使用 `gjournal` 命令来配置文件系统。通常，您可能希望对大型文件系统（如 **/usr**）进行日志记录。您需要（见下一节）预留一些空闲磁盘空间。

当一个文件系统启用日志记录时，需要一些磁盘空间来存储日志本身。存储实际数据的磁盘空间称为 *数据提供者*，而存储日志的空间称为 *日志提供者*。在对现有（非空）分区进行日志记录时，数据和日志提供者需要位于不同的分区中。而在对新分区进行日志记录时，您可以选择使用单一提供者来同时存储数据和日志。无论哪种情况，`gjournal` 命令都会将两个提供者合并以创建最终的日志文件系统。例如：

* 您希望对 **/usr** 文件系统进行日志记录，该文件系统存储在 **/dev/ad0s1f** 中（该分区已经包含数据）。
* 您在 **/dev/ad0s1g** 分区中预留了一些空闲磁盘空间。
* 使用 `gjournal`，会创建一个新的 **/dev/ad0s1f.journal** 设备，其中 **/dev/ad0s1f** 是数据提供者，**/dev/ad0s1g** 是日志提供者。然后，这个新设备将用于所有后续的文件操作。

您需要为日志提供者预留的磁盘空间量取决于文件系统的使用负载，而不是数据提供者的大小。例如，在典型的办公室桌面环境中，为 **/usr** 文件系统预留 1 GB 的日志提供者空间就足够了，而处理大量磁盘 I/O（如视频编辑）的机器可能需要更多。如果在日志空间未能提交之前就耗尽了空间，系统会发生内核恐慌。

>**注意**
>
>这里建议的日志大小，在典型的桌面使用中（例如浏览网页、文字处理和播放媒体文件）不太可能引起问题。如果您的工作负载包括密集的磁盘活动，则可以遵循以下规则来最大化可靠性：您的内存大小应该占日志提供者空间的 30%。例如，如果您的系统有 1 GB RAM，则创建一个大约 3.3 GB 的日志提供者。（将您的内容大小乘以 3.3 即可获得日志的大小）。

有关日志记录的更多信息，请阅读 [gjournal(8)](https://man.freebsd.org/cgi/man.cgi?query=gjournal&sektion=8&format=html) 手册页面。

## 3. 在安装 FreeBSD 过程中执行的步骤

### 3.1. 为日志记录预留空间

一台典型的桌面计算机通常有一块硬盘，用于存储操作系统和用户数据。可以说，sysinstall 选择的默认分区方案或多或少是合适的：一台桌面计算机不需要一个大的 **/var** 分区，而 **/usr** 分区则分配了大部分磁盘空间，因为用户数据和许多包都安装在其子目录中。

默认的分区（通过按 A 键在 FreeBSD 分区编辑器中获得的，称为 Disklabel）没有留下任何未分配的空间。每个将进行日志记录的分区都需要另一个分区来存储日志。由于 **/usr** 分区是最大的，因此稍微缩小这个分区以获得日志所需的空间是有意义的。

在我们的示例中，使用了一个 80 GB 的硬盘。以下截图显示了安装过程中 Disklabel 创建的默认分区：

![disklabel1](https://docs.freebsd.org/images/articles/gjournal-desktop/disklabel1.png)

如果这基本符合您的需求，那么调整分区以进行日志记录非常简单。只需使用箭头键将光标移到 **/usr** 分区并按 D 删除它。

现在，将光标移到屏幕顶部的磁盘名称上，按 C 创建一个新的 **/usr** 分区。这个新分区应该比原来的分区小 1 GB（如果只打算为 **/usr** 创建日志），或者小 2 GB（如果打算为 **/usr** 和 **/var** 都创建日志）。在弹出的窗口中，选择创建文件系统，并将挂载点设置为 **/usr**。

>**注意**
>
>是否应该为 **/var** 分区创建日志？通常，日志记录对于较大的分区更为合适。您可以决定是否为 **/var** 创建日志，尽管在典型的桌面环境中这样做不会造成任何问题。如果文件系统使用量较轻（桌面环境中很常见），您可能希望为其日志分配较少的磁盘空间。在我们的示例中，我们同时为 **/usr** 和 **/var** 创建日志。您当然可以根据自己的需求调整此过程。

为了尽可能简化操作，我们将使用 sysinstall 来创建日志所需的分区。然而，在安装过程中，sysinstall 会要求为每个创建的分区指定一个挂载点。在此阶段，您没有任何挂载点用于存放日志的分区，实际上您*甚至不需要挂载它们*。这些分区不是我们将来会挂载到某个位置的分区。

为避免在 sysinstall 中出现这些问题，我们将把日志分区创建为交换空间。交换空间从不挂载，sysinstall 没有问题创建任意数量的交换分区。在首次重启后，您需要编辑 **/etc/fstab** 文件，删除多余的交换空间条目。

要创建交换空间，再次使用箭头键将光标移到 Disklabel 屏幕顶部，选中磁盘名称。然后按 N，输入所需的大小（*1024M*），并从弹出的菜单中选择“交换空间”选项。对于每个要创建的日志空间重复该过程。在我们的示例中，我们创建了两个分区，分别用于 **/usr** 和 **/var** 的日志。最终结果如下所示：

![disklabel2](https://docs.freebsd.org/images/articles/gjournal-desktop/disklabel2.png)

完成分区创建后，我们建议您记录下分区名称和挂载点，以便在配置阶段轻松参考这些信息。这将有助于减少可能损坏安装的错误。下表显示了我们的示例配置的笔记：

**表 1. 分区和日志**

| 分区     | 挂载点  | 日志     |
| ------ | ---- | ------ |
| ad0s1d | /var | ad0s1h |
| ad0s1f | /usr | ad0s1g |

继续按常规方式进行安装。不过，我们建议在完全设置日志记录之前，推迟安装第三方软件（包）。

### 3.2. 第一次启动

系统将正常启动，但您需要编辑 **/etc/fstab** 文件并删除为日志创建的额外交换分区。通常，您实际使用的交换分区是带有后缀 `b` 的那个（即我们的示例中的 ad0s1b）。删除所有其他交换空间条目并重启，以便 FreeBSD 停止使用它们。

系统再次启动时，您将准备好配置日志记录。

## 4. 设置日志记录

### 4.1. 执行 `gjournal`

准备好所有必要的分区后，配置日志记录非常简单。我们需要切换到单用户模式，因此以 `root` 用户登录并输入：

```sh
# shutdown now
```

按回车键进入默认的 shell。我们需要卸载将进行日志记录的分区，在我们的示例中是 **/usr** 和 **/var**：

```sh
# umount /usr /var
```

加载日志记录所需的模块：

```sh
# gjournal load
```

现在，使用您的记录确定每个日志将使用哪个分区。在我们的示例中，**/usr** 是 **ad0s1f**，它的日志是 **ad0s1g**，而 **/var** 是 **ad0s1d**，它的日志是 **ad0s1h**。需要执行以下命令：

```sh
# gjournal label ad0s1f ad0s1g
GEOM_JOURNAL: Journal 2948326772: ad0s1f contains data.
GEOM_JOURNAL: Journal 2948326772: ad0s1g contains journal.

# gjournal label ad0s1d ad0s1h
GEOM_JOURNAL: Journal 3193218002: ad0s1d contains data.
GEOM_JOURNAL: Journal 3193218002: ad0s1h contains journal.
```

>**注意**
>
>如果某个分区的最后一个扇区已被使用，`gjournal` 会返回错误。您需要使用 `-f` 标志强制覆盖，例如：
>
>```sh
># gjournal label -f ad0s1d ad0s1h
>```
>
>由于这是全新安装，实际上不太可能有任何内容会被覆盖。 

此时，将创建两个新设备，即 **ad0s1d.journal** 和 **ad0s1f.journal**。它们分别表示我们必须挂载的 **/var** 和 **/usr** 分区。在挂载之前，我们必须先在它们上设置日志标志，并清除 Soft Updates 标志：

```sh
# tunefs -J enable -n disable ad0s1d.journal
tunefs: gjournal set
tunefs: soft updates cleared

# tunefs -J enable -n disable ad0s1f.journal
tunefs: gjournal set
tunefs: soft updates cleared
```

现在，将新设备手动挂载到各自的位置（请注意，现在我们可以使用 `async` 挂载选项）：

```sh
# mount -o async /dev/ad0s1d.journal /var
# mount -o async /dev/ad0s1f.journal /usr
```

编辑 **/etc/fstab** 文件并更新 **/usr** 和 **/var** 的条目：

```sh
/dev/ad0s1f.journal     /usr            ufs     rw,async      2       2
/dev/ad0s1d.journal     /var            ufs     rw,async      2       2
```

>**警告**
>
>请确保上述条目正确无误，否则在重启后您将无法正常启动！

最后，编辑 **/boot/loader.conf** 文件并添加以下行，以便每次启动时加载 [gjournal(8)](https://man.freebsd.org/cgi/man.cgi?query=gjournal&sektion=8&format=html) 模块：

```sh
geom_journal_load="YES"
```

恭喜！您的系统现在已启用日志记录。您可以输入 `exit` 返回到多用户模式，或者重启以测试您的配置（推荐）。在启动时，您将看到类似如下的消息：

```sh
ad0: 76293MB XEC XE800JD-00HBC0 08.02D08 at ata0-master SATA150
GEOM_JOURNAL: Journal 2948326772: ad0s1g contains journal.
GEOM_JOURNAL: Journal 3193218002: ad0s1h contains journal.
GEOM_JOURNAL: Journal 3193218002: ad0s1d contains data.
GEOM_JOURNAL: Journal ad0s1d clean.
GEOM_JOURNAL: Journal 2948326772: ad0s1f contains data.
GEOM_JOURNAL: Journal ad0s1f clean.
```

在非正常关机后，消息会略有不同，例如：

```sh
GEOM_JOURNAL: Journal ad0s1d consistent.
```

这通常意味着 [gjournal(8)](https://man.freebsd.org/cgi/man.cgi?query=gjournal&sektion=8&format=html) 使用日志提供者中的信息将文件系统恢复到一致状态。

### 4.2. 日志记录新创建的分区

虽然上述过程适用于已经包含数据的分区进行日志记录，但对于空分区进行日志记录则相对简单，因为数据和日志提供者可以存储在同一分区中。例如，假设安装了一个新磁盘，并且创建了一个新分区 **/dev/ad1s1d**，创建日志的过程就简单多了，您只需要执行：

```sh
# gjournal label ad1s1d
```

默认情况下，日志的大小将为 1 GB。您可以使用 `-s` 选项来调整它。该值可以以字节为单位给出，或者附加 `K`、`M` 或 `G` 来分别表示千字节、兆字节或吉字节。请注意，`gjournal` 不允许您创建不合适的小日志大小。

例如，要创建一个 2 GB 的日志，您可以使用以下命令：

```sh
# gjournal label -s 2G ad1s1d
```

然后，您可以在新分区上创建文件系统，并使用 `-J` 选项启用日志记录：

```sh
# newfs -J /dev/ad1s1d.journal
```

### 4.3. 将日志记录功能集成到自定义内核中

如果您不希望加载 `geom_journal` 模块，可以将其功能直接构建到内核中。编辑您的自定义内核配置文件，并确保包含以下两行：

```sh
options UFS_GJOURNAL # 注意：此项已包含在 GENERIC 中

options GEOM_JOURNAL # 需要您添加此项
```

根据 [FreeBSD 手册中的相关说明](https://docs.freebsd.org/en/books/handbook/#kernelconfig) 重新构建并重新安装您的内核。

如果您之前使用过此模块，请不要忘记从 **/boot/loader.conf** 中删除相关的 "load" 条目。

## 5. 日志记录故障排除

以下部分涵盖了有关日志记录问题的常见问题解答。

### 5.1. 在磁盘活动高峰期，我遇到了内核恐慌。这与日志记录有什么关系？

日志可能会在有机会提交（刷新）到磁盘之前就已填满。请记住，日志的大小取决于使用负载，而不是数据提供者的大小。如果您的磁盘活动较高，您需要为日志分配更大的分区。请参阅 [FreeBSD 中日志记录的理解](https://docs.freebsd.org/en/articles/gjournal-desktop/#understanding-journaling) 部分中的说明。

### 5.2. 我在配置过程中犯了一些错误，现在无法正常启动。这个问题能解决吗？

您可能忘记（或拼写错误）了 **/boot/loader.conf** 中的条目，或者 **/etc/fstab** 文件中有错误。这些问题通常很容易修复。按回车键进入默认的单用户 shell。然后找到问题的根源：

```sh
# cat /boot/loader.conf
```

如果缺少或拼写错误了 `geom_journal_load` 条目，则不会创建日志设备。手动加载模块，挂载所有分区，然后继续多用户启动：

```sh
# gjournal load

GEOM_JOURNAL: Journal 2948326772: ad0s1g contains journal.
GEOM_JOURNAL: Journal 3193218002: ad0s1h contains journal.
GEOM_JOURNAL: Journal 3193218002: ad0s1d contains data.
GEOM_JOURNAL: Journal ad0s1d clean.
GEOM_JOURNAL: Journal 2948326772: ad0s1f contains data.
GEOM_JOURNAL: Journal ad0s1f clean.

# mount -a
# exit
(boot continues)
```

另一方面，如果该条目是正确的，请检查 **/etc/fstab**。您可能会发现拼写错误或缺失的条目。在这种情况下，手动挂载所有剩余分区并继续多用户启动。

### 5.3. 我可以删除日志记录并恢复到标准的带软更新的文件系统吗？

当然可以。使用以下步骤可以撤销之前的更改。您为日志提供者创建的分区可以在之后用于其他目的（如果需要的话）。

以 `root` 用户身份登录并切换到单用户模式：

```sh
# shutdown now
```

卸载已进行日志记录的分区：

```sh
# umount /usr /var
```

同步日志：

```sh
# gjournal sync
```

停止日志记录提供者：

```sh
# gjournal stop ad0s1d.journal
# gjournal stop ad0s1f.journal
```

清除所有使用的设备上的日志元数据：

```sh
# gjournal clear ad0s1d
# gjournal clear ad0s1f
# gjournal clear ad0s1g
# gjournal clear ad0s1h
```

清除文件系统的日志记录标志，并恢复软更新标志：

```sh
# tunefs -J disable -n enable ad0s1d
tunefs: gjournal cleared
tunefs: soft updates set

# tunefs -J disable -n enable ad0s1f
tunefs: gjournal cleared
tunefs: soft updates set
```

手动重新挂载旧设备：

```sh
# mount -o rw /dev/ad0s1d /var
# mount -o rw /dev/ad0s1f /usr
```

编辑 **/etc/fstab** 并恢复到原始状态：

```sh
/dev/ad0s1f     /usr            ufs     rw      2       2
/dev/ad0s1d     /var            ufs     rw      2       2
```

最后，编辑 **/boot/loader.conf**，删除加载 `geom_journal` 模块的条目并重启。

## 6. 进一步阅读

日志记录是 FreeBSD 中一个相对较新的功能，因此它的文档还不十分完善。然而，以下一些额外的参考资料可能会对您有所帮助：

* [FreeBSD 手册中的日志记录新章节](https://docs.freebsd.org/en/books/handbook/#geom-gjournal)。
* [FreeBSD-CURRENT 邮件列表中的一篇帖子](https://lists.freebsd.org/pipermail/freebsd-current/2006-June/064043.html)，由 [gjournal(8)](https://man.freebsd.org/cgi/man.cgi?query=gjournal&sektion=8&format=html) 的开发者 Paweł Jakub Dawidek <pjd@FreeBSD.org> 提供。
* [FreeBSD general questions 邮件列表中的一篇帖子](https://lists.freebsd.org/pipermail/freebsd-questions/2008-April/173501.html)，由 Ivan Voras <ivoras@FreeBSD.org> 提供。
* [gjournal(8)](https://man.freebsd.org/cgi/man.cgi?query=gjournal&sektion=8&format=html) 和 [geom(8)](https://man.freebsd.org/cgi/man.cgi?query=geom&sektion=8&format=html) 的手册页。
