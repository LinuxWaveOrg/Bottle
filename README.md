# LinuxWaveOrg/Bottle

LinuxWave 的预编译二进制存放处。索引记录的 `url` 指向上游，但客户端不再直接去上游取 ——
4.0 起改为从本地存储取，这里的资产就是那时候被下发的字节。

这样做解决三件事：

1. 上游被攻破或出现漏洞时，不会直接波及用户 —— 下发的字节是我们验证过的那些；
2. 上游崩溃、删 release、换 CDN 时，用户不会因此装不上；
3. 用户装机量大了也不会给上游造成压力。

## 存放形态：一个包一个 Release

**资产放在 Release 里，不放在仓库文件里。** 原因是硬限制：git 单文件超过 100 MiB 会被直接
拒绝推送。Release 的单文件上限是 2 GiB，官方也明确不限制总量与下载带宽。

```
tag   = 包名
资产名 = <包名>@<版本>.<架构><扩展名>
```

客户端拼出来的地址：

```
https://github.com/LinuxWaveOrg/Bottle/releases/download/jq/jq%401.8.2.amd64
                                                     └┬┘ └──────┬──────┘
                                                     tag      资产名
```

`@` 在 URL 里要写成 `%40`。扩展名取自**上游原始文件名**，所以压缩格式不统一
（`.tar.gz` / `.tar.bz2` / `.conda` / `.zip` / 裸可执行文件），
客户端靠它选解压器。这条信息记在 `infosource` 索引的 `file` 字段里。

## 硬规则

- **不改名**：资产名与索引记录一一对应，改名等于让对应版本 404。
- **不替换**：同一版本重新构建就换一个新的版本后缀，绝不覆盖已有资产。
  上游替换同名文件这种事发生过（见 MacWave 的 `ldid` 事故），覆盖会静默改掉用户的字节。
- **先校验再上传**：每个资产上传前比对索引里的 `sha256`，不一致就不传。
- **架构必须写进文件名**：`amd64` 与 `arm64` 是同名同版本的两条记录，去掉架构就撞名。

依赖库不在这个仓库，在 [LinuxWaveOrg/Deps](https://github.com/LinuxWaveOrg/Deps)。

## 名字与项目撞车的软件包

tag 就是包名，所以这里会出现几个“看着像项目名”的 tag。它们是普通软件包，与本项目没有
任何关系，也没有特殊通道 —— 唯一要记住的是下面这张表。

| tag | 是什么 | 装出来的命令 |
| --- | --- | --- |
| `linuxwave` | [orhun/linuxwave](https://github.com/orhun/linuxwave)：Zig 写的音乐生成器，把熵变成 wav，MIT | `linuxwave` |
| `waveterm` | [Wave Terminal](https://github.com/wavetermdev/waveterm)：终端 + 图形功能的 Electron 应用 | `waveterm` |

`linuxwave` 与 LinuxWave 包管理器同名纯属巧合。两个 `linuxwave` 的差别：包管理器的命令叫
`wave`，软件包的命令叫 `linuxwave`；包管理器本体不进这个仓库（它走 `Selfupdate` 的自动更新），
所以这里不会出现“包管理器本体”的 tag。

### `wave` 这个名字保留给包管理器自己

`wave` 是 MacWave / LinuxWave 的命令名，因此**永远不做包名**。名字里带 wave 的工具按它
真实的命令命名（Wave Terminal 就是 `waveterm`）。原因很具体：安装软件包时客户端会在
`links/` 下建同名快捷方式，如果某个包叫 `wave`，它就会和包管理器自己的命令抢同一个名字。
