---
date: 2018-03-25 22:56:48
tags:
- 前端
- python
- emacs

title: 切换到 hexo
---

# 前言
我刚刚打算写一篇博文，结果发现我的 ego 坏了。回想起来我过去已经在这上面浪费了
太多的时间，干嘛不直接切换到更成熟的方案呢\_(:3」∠)_。更何况相比于 org，我现在
markdown写的更多了。于是我花了两个小时把原先的博客转换到了hexo里面。

在这里记述一下切换的经过，方便后来想要退坑的人。

# 批量从 org 转换到 markdown
我写了几行elisp来搞定这件事。

首先从网上抄来一个函数，稍微改了一下，用来获取当前目录下的所有 org 文件：
```elisp
(defun files-in-below-directory (directory)
  "List the .el files in DIRECTORY and in its sub-directories."
  ;; Although the function will be used non-interactively,
  ;; it will be easier to test if we make it interactive.
  ;; The directory will have a name such as
  ;;  "/usr/local/share/emacs/22.1.1/lisp/"
  (interactive "DDirectory name: ")
  (let (el-files-list
        (current-directory-list
         (directory-files-and-attributes directory t)))
    ;; while we are in the current directory
    (while current-directory-list
      (cond
       ;; check to see whether filename ends in '.el'
       ;; and if so, add its name to a list.
       ((equal ".org" (substring (car (car current-directory-list)) -4))
        (setq el-files-list
              (cons (car (car current-directory-list)) el-files-list)))
       ;; check whether filename is that of a directory
       ((eq t (car (cdr (car current-directory-list))))
        ;; decide whether to skip or recurse
        (if
            (equal "."
                   (substring (car (car current-directory-list)) -1))
            ;; then do nothing since filename is that of
            ;;   current directory or parent, "." or ".."
            ()
          ;; else descend into the directory and repeat the process
          (setq el-files-list
                (append
                 (files-in-below-directory
                  (car (car current-directory-list)))
                 el-files-list)))))
      ;; move to the next filename in the list; this also
      ;; shortens the list so the while loop eventually comes to an end
      (setq current-directory-list (cdr current-directory-list)))
    ;; return the filenames
    el-files-list))
```

定位到保存 org 文件的目录，做一下转换：
```elisp
(cl-loop for file in (files-in-below-directory
                      "d:/TOTALCMD/tools/emacs/home/documents/blog/blog/")
         do
         (when file
           (with-current-buffer (find-file-noselect file)
             (org-export-to-file 'gfm (concat file ".md")))))
```

我这里用的是 `org-gfm-export-to-markdown` 函数里的转换方法，这种格式的 markdown
更合我胃口。

然后我们就得到了一个混装着 `.org` 和 `.org.md` 的文件夹。

# 从org获取元信息
光有文章显然是不行的，我们还是得获取一下文章标题、日期、tag之类的东西。我写了一个
python脚本来干这件事。

```python
# 装着md和org的目录
dirPath = "d:/Temp/blog-p/"

import os
from time import strftime
now = strftime("%Y/%m/%d %H:%M:%S")

# 生成的 category
category = 'blog'

fileList = os.listdir(dirPath)
for item in fileList:
    if item.endswith('.md'):
        with open(dirPath + item, 'r+', encoding='utf-8') as out, open(dirPath + item[:-3], encoding='utf-8') as inP:
            content = out.read()
            orgAll = inP.readlines()
            global title, tags, date, keywords
            keywords = []
            for line in orgAll:
                if line.startswith('#+TITLE:'):
                    title = line
                elif line.startswith('#+DATE:'):
                    date = line
                elif line.startswith('#+TAGS:'):
                    tags = line
                elif line.startswith('#+KEYWORDS:'):
                    keywords = line[11:-1].strip().split(', ')
            title = title[8:-1].strip()
            date = date[7:-1].strip().split(' ')[0]
            tags = tags[7:-1].strip().split(', ')
            tags = list(set(tags + keywords))
            header = '---\n'
            header += 'date: ' + date.replace('-','/') + ' 00:00:00\n'
            header += 'updated: ' + now + '\n'
            header += 'tags:\n'
            for tag in tags:
                header += '- ' + tag + '\n'
            header += 'categories:\n- ' + category + '\n'
            header += 'title: ' + title + '\n'
            header += '---\n'

            out.seek(0, 0)
            out.write(header + '\n' + content)
```

你可以视情况改一下里面的几个变量。由于我以前还用过 org-page，这些文件里面有的文件
只有 `Keywords` 没有 `Tags`，正好趁这个机会合并一下。
