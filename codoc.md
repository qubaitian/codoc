[Purpose in one sentence. Split the block if that fails.](src/test.py#L3-L10)
```python
# Compute the factorial.
def factorial(n):
    # Stop the recursion.
    if n <= 1:
        return 1

    # Compute the recursive product.
    return n * factorial(n - 1)
```

## Codoc tests

[The tests define the command seam.](test/codoc-test.lisp#L1-L460)
````lisp
(defpackage #:codoc-test
  (:use #:cl #:uiop))

(in-package #:codoc-test)

(defparameter *project-root*
  (uiop:ensure-directory-pathname (uiop:getcwd)))

(defun new-test-directory ()
  (let ((directory
          (merge-pathnames
           (format nil "codoc-test-~A-~A/"
                   (get-universal-time)
                   (random 1000000))
           (uiop:temporary-directory))))
    (ensure-directories-exist directory)
    directory))

(defun set-test-file (path text)
  (ensure-directories-exist path)
  (with-open-file (stream path
                          :direction :output
                          :if-exists :supersede
                          :if-does-not-exist :create
                          :external-format :utf-8)
    (write-string text stream)))

(defun get-test-lines (path)
  (with-open-file (stream path :external-format :utf-8)
    (loop for line = (read-line stream nil nil)
          while line
          collect line)))

(defun set-codoc-process (directory &optional document)
  (let ((arguments (list (namestring (merge-pathnames "codoc" *project-root*)))))
    (when document
      (setf arguments (append arguments (list document))))
    (uiop:run-program
     arguments
     :directory directory
     :output :string
     :error-output :output)))

(defun get-codoc-result (directory document)
  (uiop:run-program
   (list (namestring (merge-pathnames "codoc" *project-root*))
         document)
   :directory directory
   :output :string
   :error-output :string
   :ignore-error-status t))

(defun assert-codoc-fails (directory document)
  (multiple-value-bind (output error status)
      (get-codoc-result directory document)
    (declare (ignore output))
    (assert (not (zerop status)))
    error))

(defun run-generates-padded-target-file ()
  (let ((directory (new-test-directory)))
    (unwind-protect
         (progn
           (set-test-file
            (merge-pathnames "document.md" directory)
            "[Example](output.txt#L2-L4)
```
alpha
beta
```
")
           (set-codoc-process directory "document.md")
           (assert
            (equal '("" "alpha" "beta" "")
                   (get-test-lines (merge-pathnames "output.txt" directory)))))
      (uiop:delete-directory-tree directory :validate t))))

(defun run-merges-out-of-order-targets ()
  (let ((directory (new-test-directory)))
    (unwind-protect
         (progn
           (set-test-file
            (merge-pathnames "document.md" directory)
            "[Later](output.txt#L4-L4)
```
later
```

[Earlier](output.txt#L1-L2)
```
first
second
```
")
           (set-codoc-process directory "document.md")
           (assert
            (equal '("first" "second" "" "later")
                   (get-test-lines (merge-pathnames "output.txt" directory)))))
      (uiop:delete-directory-tree directory :validate t))))

(defun run-uses-default-document ()
  (let ((directory (new-test-directory)))
    (unwind-protect
         (progn
           (set-test-file
            (merge-pathnames "codoc.md" directory)
            (format nil "[Default](output.txt#L1-L1)~%```~%default~%```~%"))
           (set-codoc-process directory)
           (assert
            (equal '("default")
                   (get-test-lines (merge-pathnames "output.txt" directory)))))
      (uiop:delete-directory-tree directory :validate t))))

(defun run-supports-top-level-tildes ()
  (let ((directory (new-test-directory)))
    (unwind-protect
         (progn
           (set-test-file
            (merge-pathnames "document.md" directory)
            (format nil "[Tildes](output.txt#L1-L1)~%~A~%tildes~%~A~%"
                    "~~~text"
                    "~~~"))
           (set-codoc-process directory "document.md")
           (assert
            (equal '("tildes")
                   (get-test-lines (merge-pathnames "output.txt" directory)))))
      (uiop:delete-directory-tree directory :validate t))))

(defun run-rejects-indented-fences ()
  (let ((directory (new-test-directory)))
    (unwind-protect
         (progn
           (set-test-file
            (merge-pathnames "document.md" directory)
            (format nil "[Indented](output.txt#L1-L1)~%  ```~%indented~%  ```~%"))
           (assert-codoc-fails directory "document.md")
           (assert (not (probe-file (merge-pathnames "output.txt" directory)))))
      (uiop:delete-directory-tree directory :validate t))))

(defun run-replaces-stale-target-content ()
  (let ((directory (new-test-directory)))
    (unwind-protect
         (progn
           (set-test-file
            (merge-pathnames "output.txt" directory)
            "old-one
old-two
old-three
old-four
")
           (set-test-file
            (merge-pathnames "document.md" directory)
            (format nil "[Fresh](output.txt#L2-L3)~%```~%new-one~%new-two~%```~%"))
           (set-codoc-process directory "document.md")
           (assert
            (equal '("" "new-one" "new-two")
                   (get-test-lines (merge-pathnames "output.txt" directory)))))
      (uiop:delete-directory-tree directory :validate t))))

(defun run-generates-extensionless-target ()
  (let ((directory (new-test-directory)))
    (unwind-protect
         (progn
           (set-test-file
            (merge-pathnames "document.md" directory)
            (format nil "[Script](script#L1-L1)~%```~%content~%```~%"))
           (set-codoc-process directory "document.md")
           (assert
            (equal '("content")
                   (get-test-lines (merge-pathnames "script" directory))))
           (assert (not (probe-file (merge-pathnames "script.tmp" directory)))))
      (uiop:delete-directory-tree directory :validate t))))

(defun run-rejects-overlapping-targets ()
  (let ((directory (new-test-directory)))
    (unwind-protect
         (progn
           (set-test-file
            (merge-pathnames "document.md" directory)
            "[First](output.txt#L1-L2)
```
first
second
```

[Second](output.txt#L2-L3)
```
second
third
```
")
           (assert-codoc-fails directory "document.md")
           (assert (not (probe-file (merge-pathnames "output.txt" directory)))))
      (uiop:delete-directory-tree directory :validate t))))

(defun run-rejects-long-code-blocks ()
  (let ((directory (new-test-directory)))
    (unwind-protect
         (progn
           (set-test-file
            (merge-pathnames "document.md" directory)
            "[Too long](output.txt#L1-L1)
```
first
second
```
")
           (assert-codoc-fails directory "document.md")
           (assert (not (probe-file (merge-pathnames "output.txt" directory)))))
      (uiop:delete-directory-tree directory :validate t))))

(defun run-rejects-broken-target-references ()
  (let ((directory (new-test-directory)))
    (unwind-protect
         (progn
           (set-test-file
            (merge-pathnames "document.md" directory)
            "[Broken](output.txt#L1-L1)
This is prose.
")
           (assert-codoc-fails directory "document.md")
           (assert (not (probe-file (merge-pathnames "output.txt" directory)))))
      (uiop:delete-directory-tree directory :validate t))))

(defun run-rejects-outside-targets ()
  (let* ((directory (new-test-directory))
         (name (format nil "codoc-forbidden-~A.txt" (random 1000000)))
         (outside (merge-pathnames (format nil "../~A" name) directory)))
    (unwind-protect
         (progn
           (set-test-file
            (merge-pathnames "document.md" directory)
            (format nil "[Bad](../~A#L1-L1)~%```~%bad~%```~%" name))
           (assert-codoc-fails directory "document.md")
           (assert (not (probe-file outside))))
      (when (probe-file outside)
        (delete-file outside))
      (uiop:delete-directory-tree directory :validate t))))

(defun run-rejects-outside-symbolic-links ()
  (let* ((directory (new-test-directory))
         (name (format nil "codoc-link-~A.txt" (random 1000000)))
         (outside (merge-pathnames (format nil "../~A" name) directory))
         (link (merge-pathnames "linked.txt" directory)))
    (unwind-protect
         (progn
           (set-test-file outside "original
")
           (uiop:run-program
            (list "ln" "-s" (namestring outside) (namestring link)))
           (set-test-file
            (merge-pathnames "document.md" directory)
            "[Bad](linked.txt#L1-L1)
```
changed
```
")
           (multiple-value-bind (output error status)
               (get-codoc-result directory "document.md")
             (declare (ignore output error))
             (assert (not (zerop status))))
           (assert (equal '("original") (get-test-lines outside))))
      (when (probe-file link)
        (delete-file link))
      (when (probe-file outside)
        (delete-file outside))
      (uiop:delete-directory-tree directory :validate t))))

(defun run-rejects-dangling-symbolic-links ()
  (let* ((directory (new-test-directory))
         (link (merge-pathnames "linked.txt" directory)))
    (unwind-protect
         (progn
           (uiop:run-program
            (list "ln" "-s" "missing.txt" (namestring link)))
           (set-test-file
            (merge-pathnames "document.md" directory)
            (format nil "[Bad](linked.txt#L1-L1)~%```~%changed~%```~%"))
           (assert-codoc-fails directory "document.md")
           (assert (not (probe-file (merge-pathnames "output.txt" directory)))))
      (ignore-errors (delete-file link))
      (uiop:delete-directory-tree directory :validate t))))

(defun run-reports-source-errors ()
  (let ((directory (new-test-directory)))
    (unwind-protect
         (progn
           (set-test-file
            (merge-pathnames "document.md" directory)
            "[Broken](output.txt#L1-L1)
This is prose.
")
           (multiple-value-bind (output error status)
               (get-codoc-result directory "document.md")
             (declare (ignore output))
             (assert (not (zerop status)))
             (assert (search "document.md:1" error))
             (assert (search "code fence" error :test #'char-equal))))
      (uiop:delete-directory-tree directory :validate t))))

(run-generates-padded-target-file)
(run-merges-out-of-order-targets)
(run-uses-default-document)
(run-supports-top-level-tildes)
(run-rejects-indented-fences)
(run-replaces-stale-target-content)
(run-generates-extensionless-target)
(run-rejects-overlapping-targets)
(run-rejects-long-code-blocks)
(run-rejects-broken-target-references)
(run-rejects-outside-targets)
(run-rejects-outside-symbolic-links)
(run-rejects-dangling-symbolic-links)
(run-reports-source-errors)
(format t "codoc tests passed.~%")
````

## Codoc runner

[The launcher runs Codoc with Roswell SBCL.](codoc#L1-L12)
```sh
#!/bin/sh
set -eu
project_root=$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)
if command -v ros >/dev/null 2>&1; then
  roswell=$(command -v ros)
elif [ -x /opt/homebrew/bin/ros ]; then
  roswell=/opt/homebrew/bin/ros
else
  echo "codoc: Roswell executable 'ros' was not found." >&2
  exit 1
fi
exec "$roswell" run -- --noinform --non-interactive --load "$project_root/src/codoc.lisp" --eval '(codoc:set-command)' "$@"
```

[The ASDF system names the generated Codoc source.](codoc.asd#L1-L6)
```lisp
(asdf:defsystem "codoc"
  :description "Generate source files from Markdown code blocks."
  :version "0.1.0"
  :serial t
  :components ((:file "src/codoc")))
```

[The runner implements the first generation slice.](src/codoc.lisp#L1-L600)
```lisp
(defpackage #:codoc
  (:use #:cl)
  (:export #:set-command))

(in-package #:codoc)

(defstruct target-reference
  path
  start-line
  end-line
  source-line)

(defstruct marked-code-block
  reference
  lines)

(defstruct target-file
  path
  display-path
  blocks)

(defstruct diagnostic
  source-line
  message)

(defstruct target-output
  path
  display-path
  lines)

;; The parser records source lines for actionable diagnostics.

(defun get-lines (path)
  (with-open-file (stream path :external-format :utf-8)
    (loop for line = (read-line stream nil nil)
          while line
          collect line)))

(defun new-diagnostic (source-line message)
  (make-diagnostic :source-line source-line :message message))

(defun get-diagnostic-text (diagnostic)
  (format nil "~D: ~A"
          (diagnostic-source-line diagnostic)
          (diagnostic-message diagnostic)))

(defun get-markdown-link (line)
  (let* ((text (string-trim '(#\Space #\Tab) line))
         (length (length text))
         (label-end (and (plusp length)
                         (char= (char text 0) #\[)
                         (position #\] text)))
         (destination-start
           (and label-end
                (< (+ label-end 2) length)
                (string= "](" text
                         :start1 0
                         :end1 2
                         :start2 label-end
                         :end2 (+ label-end 2))))
         (destination-end
           (and destination-start
                (position #\) text :start (+ label-end 2)))))
    (when (and destination-end
               (= destination-end (1- length)))
      (subseq text (+ label-end 2) destination-end))))

(defun get-positive-integer (text)
  (when (and (plusp (length text))
             (every #'digit-char-p text))
    (let ((number (parse-integer text)))
      (when (plusp number)
        number))))

(defun get-target-reference (destination)
  (let ((hash-position (search "#L" destination :test #'char-equal)))
    (if (null hash-position)
        (values nil nil)
        (let* ((path (subseq destination 0 hash-position))
               (range (subseq destination (1+ hash-position)))
               (separator
                 (and (>= (length range) 4)
                      (char= (char range 0) #\L)
                      (position #\- range :start 1))))
          (if (and (plusp (length path))
                   separator
                   (> separator 1)
                   (< (1+ separator) (length range))
                   (char= (char range (1+ separator)) #\L))
              (let ((start
                      (get-positive-integer
                       (subseq range 1 separator)))
                    (end
                      (get-positive-integer
                       (subseq range (+ separator 2)))))
                (if (and start end (<= start end))
                    (values
                     (make-target-reference
                      :path path
                      :start-line start
                      :end-line end)
                     nil)
                    (values nil "Target reference has an invalid line range.")))
              (values nil "Target reference must use #Lstart-Lend."))))))

(defun get-opening-fence (line)
  (when (plusp (length line))
    (let ((character (char line 0)))
      (when (find character "`~")
        (let ((length
                (loop for index from 0 below (length line)
                      while (char= character (char line index))
                      count 1)))
          (when (>= length 3)
            (values character length)))))))

(defun closing-fence-p (line character length)
  (and (>= (length line) length)
       (loop for index from 0 below length
             always (char= character (char line index)))
       (loop for index from length below (length line)
             always (find (char line index) '(#\Space #\Tab)))))

(defun get-closing-index (lines start character length)
  (loop for index from start below (length lines)
        when (closing-fence-p (nth index lines) character length)
          return index))

(defun get-marked-blocks (lines)
  (let ((blocks nil)
        (diagnostics nil)
        (index 0))
    (loop while (< index (length lines))
          do (let* ((line (nth index lines))
                    (destination (get-markdown-link line))
                    (reference nil)
                    (reference-error nil))
               (if destination
                   (multiple-value-setq (reference reference-error)
                     (get-target-reference destination))
                   (setf reference-error nil))
               (cond
                 (reference-error
                  (push
                   (new-diagnostic (1+ index) reference-error)
                   diagnostics)
                  (incf index))
                 (reference
                  (setf (target-reference-source-line reference) (1+ index))
                  (multiple-value-bind (character length)
                      (if (< (1+ index) (length lines))
                          (get-opening-fence (nth (1+ index) lines))
                          (values nil nil))
                    (if (null character)
                        (progn
                          (push
                           (new-diagnostic
                            (1+ index)
                            "Target reference must precede a code fence.")
                           diagnostics)
                          (incf index))
                        (let ((closing-index
                                (get-closing-index
                                 lines
                                 (+ index 2)
                                 character
                                 length)))
                          (if (null closing-index)
                              (progn
                                (push
                                 (new-diagnostic
                                  (+ index 2)
                                  "Code fence is not closed.")
                                 diagnostics)
                                (setf index (length lines)))
                              (progn
                                (push
                                 (make-marked-code-block
                                  :reference reference
                                  :lines (subseq lines (+ index 2) closing-index))
                                 blocks)
                                (setf index (1+ closing-index))))))))
                 (t
                  (multiple-value-bind (character length)
                      (get-opening-fence line)
                    (if (null character)
                        (incf index)
                        (let ((closing-index
                                (get-closing-index
                                 lines
                                 (1+ index)
                                 character
                                 length)))
                          (if closing-index
                              (setf index (1+ closing-index))
                              (progn
                                (push
                                 (new-diagnostic
                                  (1+ index)
                                  "Code fence is not closed.")
                                 diagnostics)
                                (setf index (length lines)))))))))))
    (values (nreverse blocks) (nreverse diagnostics))))

(declaim (ftype function get-safe-target-path))

;; We validate every target before writing generated outputs.

(defun get-target-files (blocks root)
  (let ((files (make-hash-table :test #'equal))
        (diagnostics nil))
    (dolist (block blocks)
      (let* ((reference (marked-code-block-reference block))
             (display-path (target-reference-path reference)))
        (multiple-value-bind (path error-message)
            (get-safe-target-path reference root)
          (if error-message
              (push
               (new-diagnostic
                (target-reference-source-line reference)
                (format nil "~A: ~A" display-path error-message))
               diagnostics)
              (let* ((key (namestring path))
                     (target (gethash key files)))
                (unless target
                  (setf target
                        (make-target-file
                         :path path
                         :display-path display-path
                         :blocks nil))
                  (setf (gethash key files) target))
                (push block (target-file-blocks target)))))))
    (let ((targets nil))
      (maphash
       (lambda (key target)
         (declare (ignore key))
         (setf (target-file-blocks target)
               (nreverse (target-file-blocks target)))
         (push target targets))
       files)
      (values (nreverse targets) (nreverse diagnostics)))))

(defun get-target-lines (target)
  (let* ((blocks
           (sort
            (copy-list (target-file-blocks target))
            #'<
            :key
            (lambda (block)
              (target-reference-start-line
               (marked-code-block-reference block)))))
         (end
           (reduce
            #'max
            blocks
            :key
            (lambda (block)
              (target-reference-end-line
               (marked-code-block-reference block)))))
         (output (make-list end :initial-element ""))
         (diagnostics nil)
         (previous-end nil))
    (dolist (block blocks)
      (let* ((reference (marked-code-block-reference block))
             (start (target-reference-start-line reference))
             (block-end (target-reference-end-line reference))
             (range-length (1+ (- block-end start)))
             (block-length (length (marked-code-block-lines block)))
             (overlap-p (and previous-end (<= start previous-end)))
             (long-p (> block-length range-length)))
        (when overlap-p
          (push
           (new-diagnostic
            (target-reference-source-line reference)
            (format nil "Target ranges overlap in ~A."
                    (target-file-display-path target)))
           diagnostics))
        (when long-p
          (push
           (new-diagnostic
            (target-reference-source-line reference)
            (format nil "Code block exceeds its range in ~A."
                    (target-file-display-path target)))
           diagnostics))
        (unless (or overlap-p long-p)
          (loop for line in (marked-code-block-lines block)
                for line-number from start
                do (setf (nth (1- line-number) output) line)))
        (when (or (null previous-end) (> block-end previous-end))
          (setf previous-end block-end))))
    (values output (nreverse diagnostics))))

(defun get-target-outputs (targets)
  (let ((outputs nil)
        (diagnostics nil))
    (dolist (target targets)
      (multiple-value-bind (lines target-diagnostics)
          (get-target-lines target)
        (if target-diagnostics
            (setf diagnostics (append diagnostics target-diagnostics))
            (push
             (make-target-output
              :path (target-file-path target)
              :display-path (target-file-display-path target)
              :lines lines)
             outputs))))
    (values (nreverse outputs) diagnostics)))

(defun get-path-parts (path)
  (cond
    ((zerop (length path))
     (values nil "Target path cannot be empty."))
    ((find (char path 0) '(#\/ #\\))
     (values nil "Target path must be relative."))
    ((find #\: path)
     (values nil "Target path must not contain a drive prefix."))
    (t
     (let ((parts nil)
           (start 0))
       (loop for end from 0 to (length path)
             when (or (= end (length path))
                      (char= (char path end) #\/))
               do (let ((part (subseq path start end)))
                    (cond
                      ((zerop (length part))
                       (return-from get-path-parts
                         (values nil "Target path contains an empty component.")))
                      ((string= part "..")
                       (return-from get-path-parts
                         (values nil "Target path cannot traverse upward.")))
                      ((not (string= part "."))
                       (push part parts)))
                    (setf start (1+ end))))
       (if parts
           (values (nreverse parts) nil)
           (values nil "Target path must name a file."))))))

(defun inside-root-p (path root)
  (let ((root-name (namestring (uiop:ensure-directory-pathname root)))
        (path-name (namestring path)))
    (and (not (string= root-name path-name))
         (and (>= (length path-name) (length root-name))
              (string= root-name
                       path-name
                       :start1 0
                       :end1 (length root-name)
                       :start2 0
                       :end2 (length root-name))))))

#+sbcl
(defun get-link-pathname (path)
  (let ((name (namestring path)))
    (if (and (> (length name) 1)
             (find (char name (1- (length name))) '(#\/ #\\)))
        (pathname (subseq name 0 (1- (length name))))
        path)))

#+sbcl
(defun symbolic-link-p (path)
  (let ((link-path (get-link-pathname path)))
    (handler-case
        (= (logand (sb-posix:stat-mode
                    (sb-posix:lstat (namestring link-path)))
                   #o170000)
           #o120000)
      (error () nil))))

#-sbcl
(defun symbolic-link-p (path)
  (declare (ignore path))
  nil)

#+sbcl
(defun get-symbolic-link-path (path)
  (let ((link-path (get-link-pathname path)))
    (handler-case
        (truename
         (merge-pathnames
          (sb-posix:readlink (namestring link-path))
          (uiop:pathname-directory-pathname link-path)))
      (error () nil))))

#-sbcl
(defun get-symbolic-link-path (path)
  (declare (ignore path))
  nil)

(defun get-safe-target-path (reference root)
  (let ((path (target-reference-path reference)))
    (multiple-value-bind (parts error-message)
        (get-path-parts path)
      (when error-message
        (return-from get-safe-target-path (values nil error-message)))
      (let ((current root))
        (dolist (part (butlast parts))
          (setf current (merge-pathnames (format nil "~A/" part) current))
          (let* ((link-p (symbolic-link-p current))
                 (resolved (if link-p
                               (get-symbolic-link-path current)
                               (probe-file current))))
            (when (or resolved link-p)
              (if (null resolved)
                  (return-from get-safe-target-path
                    (values nil "Target path uses a dangling symbolic link."))
                  (progn
                    (setf current (truename resolved))
                    (unless (inside-root-p current root)
                      (return-from get-safe-target-path
                        (values nil "Target path leaves the project root.")))
                    (unless (uiop:directory-pathname-p current)
                      (return-from get-safe-target-path
                        (values nil "Target path has a file parent."))))))))
        (let ((candidate
                (merge-pathnames (car (last parts)) current)))
          (let* ((link-p (symbolic-link-p candidate))
                 (resolved (if link-p
                               (get-symbolic-link-path candidate)
                               (probe-file candidate))))
            (when (or resolved link-p)
              (if (null resolved)
                  (return-from get-safe-target-path
                    (values nil "Target path uses a dangling symbolic link."))
                  (setf candidate (truename resolved)))))
          (unless (inside-root-p candidate root)
            (return-from get-safe-target-path
              (values nil "Target path leaves the project root.")))
          (when (and (probe-file candidate)
                     (uiop:directory-pathname-p candidate))
            (return-from get-safe-target-path
              (values nil "Target path names a directory.")))
          (values candidate nil))))))

(defun get-temporary-path (path)
  (loop for serial from 0
        for name = (format nil ".codoc-~A-~A"
                           (or (pathname-name path) "output")
                           serial)
        for temporary = (make-pathname
                         :name name
                         :type (pathname-type path)
                         :defaults path)
        unless (probe-file temporary)
          return temporary))

(defun shebang-p (line)
  (and (>= (length line) 2)
       (string= "#!" line :end2 2)))

(defun get-shebang-p (lines)
  (and lines (shebang-p (first lines))))

(defun set-file-mode (path executable-p)
  #+sbcl
  (when executable-p
    (sb-posix:chmod (namestring path) #o755))
  #-sbcl
  (declare (ignore path executable-p)))

;; Atomic replacement keeps existing targets intact on write errors.

(defun set-output-file (output)
  (let* ((path (target-output-path output))
         (lines (target-output-lines output))
         (temporary (get-temporary-path path)))
    (ensure-directories-exist path)
    (unwind-protect
         (progn
           (with-open-file (stream temporary
                                   :direction :output
                                   :if-exists :error
                                   :if-does-not-exist :create
                                   :external-format :utf-8)
             (dolist (line lines)
               (write-line line stream)))
           (uiop:rename-file-overwriting-target temporary path)
           (set-file-mode path (get-shebang-p lines))
           (setf temporary nil))
      (when (and temporary (probe-file temporary))
        (delete-file temporary)))))

(defun set-output-files (outputs)
  (dolist (output outputs)
    (set-output-file output)
    (format t "Generated ~A~%" (target-output-display-path output))))

(defun set-diagnostics (document-path diagnostics)
  (dolist (diagnostic diagnostics)
    (format *error-output* "codoc: ~A:~D: ~A~%"
            (namestring document-path)
            (diagnostic-source-line diagnostic)
            (diagnostic-message diagnostic))))

(defun get-document-path (document-name root)
  (merge-pathnames document-name root))

(defun set-command-run (arguments)
  (if (> (length arguments) 1)
      (progn
        (format *error-output*
                "codoc: expected zero or one document path.~%")
        1)
      (handler-case
          (let* ((root
                   (uiop:ensure-directory-pathname
                    (truename (uiop:getcwd))))
                 (document-name (or (first arguments) "codoc.md"))
                 (document-path (get-document-path document-name root))
                 (lines (get-lines document-path)))
            (multiple-value-bind (blocks parse-diagnostics)
                (get-marked-blocks lines)
              (multiple-value-bind (targets path-diagnostics)
                  (get-target-files blocks root)
                (multiple-value-bind (outputs output-diagnostics)
                    (get-target-outputs targets)
                  (let ((diagnostics
                          (append parse-diagnostics
                                  path-diagnostics
                                  output-diagnostics)))
                    (if diagnostics
                        (progn
                          (set-diagnostics document-path diagnostics)
                          1)
                        (progn
                          (set-output-files outputs)
                          0)))))))
        (error (condition)
          (format *error-output* "codoc: ~A~%" condition)
          1))))

(defun set-command ()
  (uiop:quit (set-command-run (uiop:command-line-arguments))))
```

[Purpose in one sentence. Split the block if that fails.](src/test.py#L14-L22)
```python
# Compute the factorial.
def factorial(n):
    # Stop the recursion.
    if n <= 1:
        return 1

    # Compute the recursive product.
    return n * factorial(n - 1)
```
