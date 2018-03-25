---
date: 2016/09/21 00:00:00
updated: 2018/03/25 22:48:19
tags:
- emacs
categories:
- blog
title: 执行当前 buffer 中的源代码
---

- [正文](#sec-)
- [TIPS](#sec-)


# 正文<a id="sec-"></a>

Xah Lee 写过一个[函数](http://ergoemacs.org/emacs/elisp_run_current_file.html)， 可以执行当前 buffer 中的源代码。如果是解释执行的程序，则调用相应的 编译器/解释器，如果是编译执行的，则先编译代码，然后执行编译之后的程序。但是这个函数有一些缺陷， 如果代码有错误导致编译无法完成，这个函数仍然会尝试运行程序。我对它做了一些修改，如果编译不成功， 则不会运行程序；如果编译成功，则会在新的 buffer 中运行程序，这个 buffer 会在 `*compilation*` 的窗口中显示，覆盖原来的 buffer。代码如下（我还加上的编译 c++的支持）：

```emacs-lisp
;;; -----------liu233w/run-current-file-------------------
(defun liu233w/run-current-file ()
  "Execute the current file.
For example, if the current buffer is the file x.py, then it'll call「python x.py」in a shell.
The file can be Emacs Lisp, PHP, Perl, Python, Ruby, JavaScript, Bash, Ocaml, Visual Basic, TeX, Java, Clojure.
File suffix is used to determine what program to run.

If the file is modified or not saved, save it automatically before run.

URL `http://ergoemacs.org/emacs/elisp_run_current_file.html'
version 2016-01-28"
  (interactive)
  (let (
        (-suffix-map
         ;; (‹extension› . ‹shell program name›)
         `(
           ("php" . "php")
           ("pl" . "perl")
           ("py" . "python")
           ("py3" . ,(if (string-equal system-type "windows-nt") "python.exe" "python3"))
           ("rb" . "ruby")
           ("go" . "go run")
           ("js" . "node") ; node.js
           ("sh" . "bash")
           ("clj" . "java -cp /home/xah/apps/clojure-1.6.0/clojure-1.6.0.jar clojure.main")
           ("rkt" . "racket")
           ("ml" . "ocaml")
           ("vbs" . "cscript")
           ("tex" . "pdflatex")
           ("latex" . "pdflatex")
           ("java" . "javac")
           ("cpp" . ,(or (executable-find "myclang")
                         (executable-find "clang")
                         "g++"))
           ;; ("pov" . "/usr/local/bin/povray +R2 +A0.1 +J1.2 +Am2 +Q9 +H480 +W640")
           ))

        -fname
        -fSuffix
        -prog-name
        -cmd-str)

    (when (null (buffer-file-name)) (save-buffer))
    (when (buffer-modified-p) (save-buffer))

    (setq -fname (buffer-file-name))
    (setq -fSuffix (file-name-extension -fname))
    (setq -prog-name (cdr (assoc -fSuffix -suffix-map)))
    (setq -cmd-str (concat -prog-name " \""   -fname "\""))

    (cond
     ((string-equal -fSuffix "el") (load -fname))
     ((string-equal -fSuffix "java")
      (liu233w//compile-and-run
       -cmd-str
       (format "java %s" (file-name-sans-extension (file-name-nondirectory -fname)))))
     ((string-equal -fSuffix "cpp")
      (liu233w//compile-and-run
       (format "%s --std=c++11" -cmd-str)
       "a"))
     (t (if -prog-name
            (progn
              (message "Running…")
              (async-shell-command -cmd-str "*liu233w/run-current-file output*" ))
          (message "No recognized program file suffix for this file."))))))

(defun liu233w//compile-and-run (cmp-cmd run-cmd)
  "Run cmp-cmd, if success, then run run-cmd and print the result.
Or just print the error message.

When it's compiling a file, this function may cause error behavior."
  (setf liu233w//compile-status (cons run-cmd compile-command))
  (compile cmp-cmd)
  )

(defvar liu233w//compile-status nil "doc")

(defun liu233w//run-after-compile (buffer string)
  (when (and
         (string-match "compilation" (buffer-name buffer))
         (string-match "finished" string)
         liu233w//compile-status)
    ;; 防止 liu233w/bury-compile-buffer-if-successful 删除 compilation buffer
    ;; 这样 async-shell-command 命令就能覆盖 compilation buffer 而不是源代码的
    ;; buffer 了。
    (with-current-buffer "*compilation*"
      (insert "warning LOL"))
    (async-shell-command (car liu233w//compile-status)
                         "*liu233w/run-current-file output*")
    (setf compile-command (cdr liu233w//compile-status)
          liu233w//compile-status nil)))
(add-hook 'compilation-finish-functions
          #'liu233w//run-after-compile)
;;; END -----------liu233w/run-current-file-------------------
```

理论上这个函数没法在编译还没完成的时候运行，但我也没测试过。

另外，我还找到了另一段代码，可以在编译结果没有 error 或 warning 的时候自动关闭 `*compilation*` buffer。不过如果想要把这段代码搭配 `run-current-file` 运行的话需要把函数加到 hook 的最后，而不 是前面。我的代码也是通过在 `*compilation*` buffer 写入 warning 的文字来阻止这段代码关闭相应的 窗口从而令程序的运行结果显示在原来显示编译状态的位置的。

```emacs-lisp
;;; from http://stackoverflow.com/questions/11043004/emacs-compile-buffer-auto-close
(defun liu233w/bury-compile-buffer-if-successful (buffer string)
  "Bury a compilation buffer if succeeded without warnings "
  (if (and
       (string-match "compilation" (buffer-name buffer))
       (string-match "finished" string)
       (not
        (with-current-buffer buffer
          (goto-char (point-min))
          (search-forward "warning" nil t))))
      (run-with-timer 1 nil
                      (lambda (buf)
                        (bury-buffer buf)
                        (delete-window (get-buffer-window buf)))
                      buffer)))
(add-hook 'compilation-finish-functions
          #'liu233w/bury-compile-buffer-if-successful
          t)
```

# TIPS<a id="sec-"></a>

-   `add-hook` 函数有一个可选形参 `:append` ，为 t 时相应的函数将加到 hook 的最后，而不是最前面。
