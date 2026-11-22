---
title: SQLite 恢复后，怎么确认读到的是正确的一版资料
date: 2026-11-23 02:52:00 +0800
categories:
- 技术
- 工具
tags:
- SQLite
- 备份
author: jake-tao
math: false
mermaid: false
comments: true
---

数据库能正常打开，条目数量也一样，恢复的却可能是昨天的资料。以一个保存条目网址的 SQLite 目录为例，恢复后应核对两件事：文件和表结构完整，以及读者能找到这一版应该有的条目。

## 备份时保存一份条目清单

用 SQLite 的备份接口得到一致的快照。从这份快照导出清单，再把两份文件用同一个发布日期或版本号标记。事后需要同时恢复正文时，清单还可以记录条目修订号或摘要哈希。公开保存的清单只收录必要的公开字段。

## 恢复后先用只读连接核对

下面的 Python 程序构造两个模拟数据库。它们的条数一样，结构也完整，但旧版本中的一个条目和清单对不上。

```python
import sqlite3
from contextlib import closing
from pathlib import Path
from tempfile import TemporaryDirectory


def build_fixture(path, records):
    with closing(sqlite3.connect(path)) as db:
        db.execute('CREATE TABLE resource (slug TEXT PRIMARY KEY NOT NULL)')
        db.executemany('INSERT INTO resource VALUES (?)', [(slug,) for slug in records])
        db.commit()


def verify(path, expected):
    uri = path.resolve().as_uri() + '?mode=ro'
    with closing(sqlite3.connect(uri, uri=True)) as db:
        assert db.execute('PRAGMA integrity_check').fetchall() == [('ok',)]
        assert db.execute('PRAGMA foreign_key_check').fetchone() is None
        return [slug for slug, in db.execute('SELECT slug FROM resource ORDER BY slug')] == expected


with TemporaryDirectory() as temp:
    root = Path(temp)
    expected = ['edit-an-audio-clip', 'getting-started']
    build_fixture(root / 'intended.db', expected)
    build_fixture(root / 'stale.db', ['basic-recording', 'getting-started'])
    assert verify(root / 'intended.db', expected)
    assert not verify(root / 'stale.db', expected)
    print('Same row count, stale catalog detected.')
```

看到输出后，可以试着把旧数据库里的条目替换成清单预期的那一项，再运行代码。实际恢复时，把示例中的 `expected` 换成随目标快照保存的清单。还原之后可以先开预览服务，请求首页、一个分类页和一个条目，确认三个页面的标题和链接都正确。收到 200 时也要看看返回内容，避免误把统一的首页回退当成恢复成功。

[资料星球公开站点地图](https://resources.packscena.com/sitemap.xml)可以用来观察一份对外网址清单的结构。对于自己管理的站点，恢复前后可以把公开地图与对应版本的条目清单交叉核对。

参考：[SQLite Online Backup API](https://www.sqlite.org/backup.html)、[SQLite PRAGMA 说明](https://www.sqlite.org/pragma.html)。

作者 Jake Tao，维护文中链接的资料目录。本文由 AI 协助整理，示例数据为虚构。
