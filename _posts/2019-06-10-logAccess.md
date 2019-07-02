---
layout: post
title:  "日志写入配置"
crawlertitle: "How to log"
summary: ""
date:   2019-06-10 00:09:47
categories: note
tags: 'linux'
author: yy
---

在配置task的时候用了crontab和supervisor两种方式实现，supervisor的用户配的nobody,所以出现两个日志写入问题。之前php也遇到过这个问题，因为当时日志封装用的fle_put_content，可以在代码层写日志的时候直接修改权限，umask。也可以在代码执行层保证用户统一。

1. Umask

Umask: 在linux系统，新建文件默认权限为666，新建目录默认权限为777。但是要"减去" umask的值，umask的值可以使用umask命令看到，一般情况下,root用户的为022。所以不改的情况下，一般程序创建的文件夹权限是755，文件权限644。

下面程序设置成0，是针对当前process( php or python)创建的文件强制设置成666（rw）， 所以多线程的情况下用的时候要注意。

- php file_put_contents

  ```php
  $strLog = date('Y-m-d H:i:s') . "\t" . $strLog . "\n";
  $strPath = self::path($strSubdir);
  $strFile = $strPath. '/' . $strFile . '.log';
  if (!is_dir($strPath))
  {
    $old = umask(0);
    mkdir($strPath, 0777, true);
    umask($old);#这里在设置成原来的旧值
  }
  
  return file_put_contents($strFile, $strLog, FILE_APPEND);
  ```

- python logging

  ```python
  os.umask(0)##这个地方更改权限，恢复的触发点咩有写
  if not os.path.exists(LOG_ROOT):
    os.makedirs(LOG_ROOT)
  cfg_file = os.path.join(os.path.split(os.path.realpath(__file__))[0], 'logging.yaml')
  cfg = yaml.load(open(cfg_file, 'r'))
  if 'handlers' in cfg:
    for h, k_cfg in cfg['handlers'].items():
        if k_cfg.has_key('filename'):
            k_cfg['filename'] = os.path.join(LOG_ROOT, k_cfg['filename'])
  logging.config.dictConfig(cfg)
  ```

  

2.用户一致

crontab -e的方式创建的crontab的用户是执行这个命令的当前用户。

如果想配置一些其他用户，/etc/crontab,支持user配置。把crontab也配置成nobody即可。