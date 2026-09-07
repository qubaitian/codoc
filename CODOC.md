# Implementation and tests for codoc

This document is the maintenance source for codoc's implementation, build scripts, and tests.  
Each purpose link maps the following code block to a target range in a file.  

## Bootstrap and maintenance

With an existing codoc binary, place it next to this document to restore the source and tests.  
Run the following commands in that directory to generate, rebuild, regenerate, and test.  
Rebuilding and running tests requires SBCL.  

```shell
./codoc get
sbcl --script src/build.lisp
sbcl --script tests/codoc-tests.lisp
```

Edit the mapped blocks in this document when changing the implementation.  
When a code block's line count changes, update its target range and later ranges in the same file.  
Running `./codoc get` regenerates the four Lisp files in `src/` and `tests/` from this document.  
Edits to generated files are overwritten on the next generation.  
The shell block above has no purpose link, so it is not written to a file.  

## Packages and data structures

The public entry points run commands, generate files, and delete mapped files.  

### Public interface

[Define the codoc package and its public entry points.](src/codoc.lisp#L1-L5)  
```commonlisp
(defpackage :codoc
  (:use :cl)
  (:export :set-main :set-document :del-document))

(in-package :codoc)
```

### Mapped block

[Store a mapped block's target range, code lines, and document location.](src/codoc.lisp#L7-L11)  
```commonlisp
(defstruct (patch (:constructor new-patch)
                  (:conc-name get-patch-)
                  (:predicate if-patch-p)
                  (:copier nil))
  path start end lines location)
```

### Target file

[Store a target file's mapped blocks and the text to write.](src/codoc.lisp#L13-L17)  
```commonlisp
(defstruct (target (:constructor new-target)
                   (:conc-name get-target-)
                   (:predicate if-target-p)
                   (:copier nil))
  path patches text)
```

## Text reading

Text is read as UTF-8, then split into lines for the parser.  

### Trim edge whitespace

[Trim spaces, tabs, and carriage returns from both ends of text.](src/codoc.lisp#L19-L20)  
```commonlisp
(defun get-trimmed (text)
  (string-trim '(#\Space #\Tab #\Return) text))
```

### Recognize indent spaces

[Return whether a character is a space.](src/codoc.lisp#L22-L23)  
```commonlisp
(defun if-space-p (character)
  (char= character #\Space))
```

### Read a file

[Read a file's UTF-8 text.](src/codoc.lisp#L25-L29)  
```commonlisp
(defun get-file-text (path)
  (with-open-file (stream path :external-format :utf-8)
    (let* ((text (make-string (file-length stream)))
           (length (read-sequence text stream)))
      (subseq text 0 length))))
```

### Split text into lines

[Split text on newlines and strip trailing carriage returns.](src/codoc.lisp#L31-L36)  
```commonlisp
(defun get-text-lines (text)
  (loop for start = 0 then (1+ end)
        while (< start (length text))
        for end = (position #\Newline text :start start)
        collect (string-right-trim '(#\Return) (subseq text start end))
        while end))
```

## Fences and purpose links

Purpose links supply target locations.  
Code fences bound the content to write.  

### Recognize a code fence

[Read a fence's marker, width, trailing text, and indent.](src/codoc.lisp#L38-L49)  
```commonlisp
(defun get-fence (line)
  (let* ((indent (or (position-if-not #'if-space-p line)
                     (length line)))
         (marker (and (< indent (length line)) (char line indent))))
    (when (and (<= indent 3) (member marker '(#\` #\~)))
      (let* ((end (loop for index from indent below (length line)
                        while (char= (char line index) marker)
                        finally (return index)))
             (width (- end indent))
             (tail (subseq line end)))
        (when (>= width 3)
          (values marker width tail indent))))))
```

### Parse a line number

[Parse a one-based decimal line number.](src/codoc.lisp#L51-L57)  
```commonlisp
(defun get-line-number (text location)
  (unless (and (plusp (length text)) (every #'digit-char-p text))
    (error "~A: invalid line number ~S." location text))
  (let ((number (parse-integer text)))
    (unless (plusp number)
      (error "~A: line numbers must start at 1." location))
    number))
```

### Canonicalize a target path

[Resolve a target path to a canonical path for grouping by file.](src/codoc.lisp#L59-L85)  
```commonlisp
(defun get-canonical-path (name base)
  ;; Resolve existing directory aliases before grouping patches by file.
  (let* ((path (merge-pathnames (parse-namestring name) base))
         (directory (pathname-directory path))
         (resolved (make-pathname :directory '(:absolute)
                                  :defaults path :name nil :type nil)))
    (when (wild-pathname-p path)
      (error "Wildcard target paths are not supported: ~A." name))
    (dolist (part (rest directory))
      (setf resolved
            (if (eq part :up)
                (make-pathname :directory
                               (if (cdr (pathname-directory resolved))
                                   (butlast (pathname-directory resolved))
                                   '(:absolute))
                               :defaults resolved)
                (merge-pathnames (make-pathname :directory (list :relative part))
                                 resolved)))
      (setf resolved (or (probe-file resolved) resolved)))
    (let* ((result (make-pathname :name (pathname-name path)
                                 :type (pathname-type path)
                                 :version (pathname-version path)
                                 :defaults resolved))
           (result (or (probe-file result) result)))
      (unless (pathname-name result)
        (error "Target must name a file: ~A." name))
      result)))
```

### Parse a purpose link

[Read a mapped block's target range from a local purpose link.](src/codoc.lisp#L87-L115)  
```commonlisp
(defun get-link-patch (line base location)
  (let* ((text (get-trimmed line))
         (separator (search "](" text)))
    (when (and separator (plusp (length text))
               (char= (char text 0) #\[)
               (char= (char text (1- (length text))) #\)))
      (let* ((destination (subseq text (+ separator 2) (1- (length text))))
             (destination
               (if (and (> (length destination) 1)
                        (char= (char destination 0) #\<)
                        (char= (char destination (1- (length destination))) #\>))
                   (subseq destination 1 (1- (length destination)))
                   destination))
             (anchor (search "#L" destination :from-end t)))
        (when (and anchor
                   (not (find #\: destination :end anchor)))
          (let* ((name (subseq destination 0 anchor))
                 (range (subseq destination (+ anchor 2)))
                 (dash (search "-L" range))
                 (start (get-line-number (subseq range 0 dash) location))
                 (end (if dash
                          (get-line-number (subseq range (+ dash 2)) location)
                          start)))
            (when (zerop (length name))
              (error "~A: target file is missing." location))
            (when (> start end)
              (error "~A: target range is reversed." location))
            (new-patch :path (get-canonical-path name base)
                       :start start :end end :location location)))))))
```

## Document scanning

The scanner remembers the latest nonempty line outside a fence.  
The scanner collects code inside a fence.  
The next three blocks continue one function by target line number.  

### Initialize scan state

[Initialize document scan state and recognize fences line by line.](src/codoc.lisp#L117-L131)  
```commonlisp
(defun get-document-patches (document)
  (let ((base (make-pathname :name nil :type nil :defaults document))
        (previous "")
        (previous-number 0)
        (marker nil)
        (width 0)
        (indent 0)
        (patch nil)
        (body nil)
        (patches nil))
    (loop for line in (get-text-lines (get-file-text document))
          for number from 1
          do (multiple-value-bind (next-marker next-width tail next-indent)
                 (get-fence line)
               (cond
```

### Collect and validate code

[Collect fenced code and check its line count when the fence closes.](src/codoc.lisp#L132-L153)  
```commonlisp
                 (marker
                  (if (and (eql marker next-marker)
                           (>= next-width width)
                           (string= (get-trimmed tail) ""))
                      (progn
                        (when patch
                          (setf (get-patch-lines patch) (nreverse body))
                          (unless (= (length (get-patch-lines patch))
                                     (1+ (- (get-patch-end patch)
                                            (get-patch-start patch))))
                            (error "~A: expected ~D code lines, found ~D."
                                   (get-patch-location patch)
                                   (1+ (- (get-patch-end patch)
                                          (get-patch-start patch)))
                                   (length (get-patch-lines patch))))
                          (push patch patches))
                        (setf marker nil patch nil body nil previous ""))
                      (when patch
                        ;; Markdown removes up to the opening fence's indentation.
                        (let ((spaces (or (position-if-not #'if-space-p line)
                                          (length line))))
                          (push (subseq line (min indent spaces)) body)))))
```

### Attach links and finish scanning

[Attach a purpose link to a new fence and reject an unclosed mapped block at the end of the scan.](src/codoc.lisp#L154-L165)  
```commonlisp
                 ((and next-marker
                       (not (and (char= next-marker #\`) (find #\` tail))))
                  (setf marker next-marker width next-width indent next-indent
                        patch (get-link-patch previous base
                                              (format nil "~A:~D"
                                                      document previous-number))))
                 ((plusp (length (get-trimmed line)))
                  (setf previous line previous-number number)))))
    (when (and marker patch)
      (error "~A: mapped code block has no closing fence."
             (get-patch-location patch)))
    (nreverse patches)))
```

## Target file generation

Target files are deleted and recreated only after every mapping validates.  
Gaps between target ranges become blank lines.  

### Assemble file text

[Assemble a target file's full text from mapped ranges.](src/codoc.lisp#L167-L176)  
```commonlisp
(defun new-target-text (target)
  (let* ((patches (get-target-patches target))
         (size (get-patch-end (car (last patches))))
         (result (make-array size :initial-element "")))
    (dolist (patch patches)
      (loop for line in (get-patch-lines patch)
            for index from (1- (get-patch-start patch))
            do (setf (aref result index) line)))
    (with-output-to-string (stream)
      (loop for line across result do (write-line line stream)))))
```

### Prepare a write plan

[Create a write plan grouped by file with non-overlapping ranges.](src/codoc.lisp#L178-L197)  
```commonlisp
(defun new-write-plan (patches)
  (let ((targets nil))
    (dolist (patch patches)
      (let ((target (find (get-patch-path patch) targets
                          :key #'get-target-path :test #'equal)))
        (unless target
          (setf target (new-target :path (get-patch-path patch)))
          (push target targets))
        (push patch (get-target-patches target))))
    (dolist (target targets)
      (setf (get-target-patches target)
            (sort (get-target-patches target) #'< :key #'get-patch-start))
      (loop for (left right) on (get-target-patches target)
            while right
            when (>= (get-patch-end left) (get-patch-start right))
              do (error "~A: target range overlaps ~A in ~A."
                        (get-patch-location right) (get-patch-location left)
                        (get-target-path target)))
      (setf (get-target-text target) (new-target-text target)))
    (nreverse targets)))
```

### Delete old targets

[Delete existing target files in a write plan.](src/codoc.lisp#L199-L202)  
```commonlisp
(defun del-target-files (plan)
  (dolist (target plan)
    (when (probe-file (get-target-path target))
      (delete-file (get-target-path target)))))
```

### Delete mapped files

[Validate a document, then delete its mapped target files.](src/codoc.lisp#L204-L210)  
```commonlisp
(defun del-document (filename)
  "Validate a document, then delete its mapped target files."
  (let* ((document (truename filename))
         (patches (get-document-patches document))
         (plan (new-write-plan patches)))
    (del-target-files plan)
    (values (length patches) (length plan))))
```

### Generate target files

[Validate a document, then recreate its target files from the write plan.](src/codoc.lisp#L212-L224)  
```commonlisp
(defun set-document (filename)
  "Validate a document, then recreate its target files from mapped blocks."
  (let* ((document (truename filename))
         (patches (get-document-patches document))
         (plan (new-write-plan patches)))
    (del-target-files plan)
    (dolist (target plan)
      (ensure-directories-exist (get-target-path target))
      (with-open-file (stream (get-target-path target) :direction :output
                             :if-exists :error :if-does-not-exist :create
                             :external-format :utf-8)
        (write-string (get-target-text target) stream)))
    (values (length patches) (length plan))))
```

## Command-line behavior

The command line requires `del` or `get`.  
Both commands read `CODOC.md` in the current directory by default.  
`del` deletes all mapped target files.  
`get` deletes mapped target files, then recreates them from the document.  
A greeting example is created only when `get` omits the document argument and the default document is missing.  

### Create the default document

[Create the default document with a greeting example mapping.](src/codoc.lisp#L226-L244)  
```commonlisp
(defun new-default-document ()
  (with-open-file (stream "CODOC.md" :direction :output
                         :if-exists :error :if-does-not-exist :create
                         :external-format :utf-8)
    (format stream "~{~A~%~}"
            '("# Hello from codoc"
              ""
              "[Return a greeting.](examples/hello.lisp#L1-L2)"
              "```commonlisp"
              "(defun get-greeting ()"
              "  \"Hello from codoc.\")"
              "```"
              ""
              "Run codoc get to regenerate the greeting example.  "
              ""
              "```shell"
              "codoc get"
              "sbcl --noinform --load examples/hello.lisp --eval '(write-line (get-greeting))' --quit"
              "```"))))
```

### Handle command arguments

[Run the action for the command arguments and return an exit code.](src/codoc.lisp#L246-L282)
```commonlisp
(defun set-main (arguments)
  (handler-case
      (cond
        ((equal arguments '("--help"))
         (format t "~{~A~%~}"
                 '("Usage: codoc <del|get> [MARKDOWN-FILE]"
                   "Default: ./CODOC.md"
                   "del deletes all mapped target files."
                   "get deletes mapped target files, then recreates them from the document."
                   "get creates a greeting example if ./CODOC.md is missing."))
         0)
        ((or (null arguments)
             (> (length arguments) 2)
             (and (plusp (length (first arguments)))
                  (char= (char (first arguments) 0) #\-))
             (not (member (first arguments) '("del" "get") :test #'equal)))
         (format *error-output* "Usage: codoc <del|get> [MARKDOWN-FILE]~%")
         2)
        (t
         (let ((command (first arguments))
               (filename (or (second arguments) "CODOC.md")))
           (when (and (equal command "get")
                      (null (second arguments))
                      (not (probe-file "CODOC.md")))
             (new-default-document))
           (multiple-value-bind (blocks files)
               (if (equal command "del")
                   (del-document filename)
                   (set-document filename))
             (if (equal command "del")
                 (format t "Deleted ~D mapped file~:P.~%" files)
                 (format t "Wrote ~D code block~:P to ~D file~:P.~%"
                         blocks files))))
         0))
    (error (condition)
      (format *error-output* "codoc: ~A~%" condition)
      1)))
```

## Scripts and the binary

The script entry and the binary entry both call `set-main`.  
The build script saves an executable image with SBCL.  

### Script entry

[Load the core implementation and exit the script with the command result.](src/cli.lisp#L1-L2)  
```commonlisp
(load (merge-pathnames "codoc.lisp" *load-truename*))
(sb-ext:exit :code (codoc:set-main (cdr sb-ext:*posix-argv*)))
```

### Load the build sources

[Load the core implementation needed for the build.](src/build.lisp#L1)  
```commonlisp
(load (merge-pathnames "codoc.lisp" *load-truename*))
```

### Binary entry

[Exit the binary with the command result.](src/build.lisp#L3-L4)  
```commonlisp
(defun codoc::set-cli-main ()
  (sb-ext:exit :code (codoc:set-main (cdr sb-ext:*posix-argv*))))
```

### Save the executable

[Save the current Lisp image as the codoc binary.](src/build.lisp#L6-L10)  
```commonlisp
(sb-ext:save-lisp-and-die (merge-pathnames
                         "../codoc"
                         (make-pathname :name nil :type nil :defaults *load-truename*))
                        :toplevel #'codoc::set-cli-main
                        :executable t :save-runtime-options t)
```

## Test infrastructure

Tests use a private temporary directory for documents and generated files.  

### Test runtime

[Initialize the test package, check counter, and path to the binary under test.](tests/codoc-tests.lisp#L1-L14)  
```commonlisp
(require :asdf)
(load (merge-pathnames "../src/codoc.lisp" *load-truename*))

(defpackage :codoc-tests
  (:use :cl))

(in-package :codoc-tests)

(defparameter *root* nil)
(defparameter *checks* 0)
(defparameter *executable*
  (namestring (truename (merge-pathnames
                         "../codoc"
                         (make-pathname :name nil :type nil :defaults *load-truename*)))))
```

### Locate a test file

[Return a test file path inside the temporary directory.](tests/codoc-tests.lisp#L16-L17)  
```commonlisp
(defun get-path (name)
  (merge-pathnames name *root*))
```

### Build multiline text

[Create multiline test text that ends with a newline.](tests/codoc-tests.lisp#L19-L20)  
```commonlisp
(defun new-lines (&rest lines)
  (format nil "~{~A~%~}" lines))
```

### Write a test file

[Write test text to a named file in the temporary directory.](tests/codoc-tests.lisp#L22-L28)  
```commonlisp
(defun set-file (name text)
  (let ((path (get-path name)))
    (ensure-directories-exist path)
    (with-open-file (stream path :direction :output :if-exists :supersede
                                :if-does-not-exist :create :external-format :utf-8)
      (write-string text stream))
    path))
```

### Compare results

[Check that an actual value equals an expected value.](tests/codoc-tests.lisp#L30-L33)  
```commonlisp
(defun test-equal (expected actual)
  (incf *checks*)
  (unless (equal expected actual)
    (error "Expected ~S, got ~S." expected actual)))
```

### Check file contents

[Check the full text of a test file.](tests/codoc-tests.lisp#L35-L36)  
```commonlisp
(defun test-file (name expected)
  (test-equal expected (codoc::get-file-text (get-path name))))
```

### Check a rejection reason

[Check that an invalid document's error message contains a given fragment.](tests/codoc-tests.lisp#L38-L45)  
```commonlisp
(defun test-rejection (text fragment)
  (set-file "invalid.md" text)
  (let ((message (handler-case
                     (progn (codoc:set-document (get-path "invalid.md")) nil)
                   (error (condition) (princ-to-string condition)))))
    (incf *checks*)
    (unless (and message (search fragment message))
      (error "Expected error containing ~S, got ~S." fragment message))))
```

## File generation tests

These tests cover mapped ranges, cleanup of old content, and target file creation.  

### Ranges and regeneration

[Verify that out-of-order mappings generate the same file by target line number.](tests/codoc-tests.lisp#L47-L57)  
```commonlisp
(defun test-ranges ()
  (set-file "existing.lisp" (new-lines "one" "two" "three" "four" "five"))
  (set-file "docs/input.md"
            (new-lines "[Later](../existing.lisp#L4-L4)" "```lisp" "FOUR" "```"
                       "[Earlier](../existing.lisp#L2-L3)" "" "~~~lisp"
                       "TWO" "THREE" "~~~~"))
  (test-equal '(2 1)
              (multiple-value-list (codoc:set-document (get-path "docs/input.md"))))
  (test-file "existing.lisp" (new-lines "" "TWO" "THREE" "FOUR"))
  (codoc:set-document (get-path "docs/input.md"))
  (test-file "existing.lisp" (new-lines "" "TWO" "THREE" "FOUR")))
```

### Clean old content

[Verify that recreation affects only mapped target files.](tests/codoc-tests.lisp#L59-L77)  
```commonlisp
(defun test-clean-targets ()
  (set-file "clean/output.txt" (new-lines "old" "old" "old" "old" "old" "old"))
  (set-file "clean/unmapped.txt" (new-lines "keep"))
  ;; A hard link retains the old file when its mapped name is deleted.
  (test-equal 0
              (sb-ext:process-exit-code
               (sb-ext:run-program "/bin/ln"
                                   (list (namestring (get-path "clean/output.txt"))
                                         (namestring (get-path "clean/original.txt")))
                                   :wait t)))
  (set-file "clean.md"
            (new-lines "[Last](clean/output.txt#L4)" "```" "last" "```"
                       "[First](clean/output.txt#L2)" "```" "first" "```"
                       "[Unmapped](clean/unmapped.txt)" "```" "ignored" "```"))
  (test-equal '(2 1)
              (multiple-value-list (codoc:set-document (get-path "clean.md"))))
  (test-file "clean/output.txt" (new-lines "" "first" "" "last"))
  (test-file "clean/original.txt" (new-lines "old" "old" "old" "old" "old" "old"))
  (test-file "clean/unmapped.txt" (new-lines "keep")))
```

### Delete mapped files

[Verify that deletion removes mapped target files and leaves unmapped files.](tests/codoc-tests.lisp#L79-L89)  
```commonlisp
(defun test-del-document ()
  (set-file "mapped.txt" (new-lines "old"))
  (set-file "keep.txt" (new-lines "keep"))
  (set-file "del.md"
            (new-lines "[Mapped](mapped.txt#L1)" "```" "new" "```"))
  (codoc:set-document (get-path "del.md"))
  (test-file "mapped.txt" (new-lines "new"))
  (test-equal '(1 1)
              (multiple-value-list (codoc:del-document (get-path "del.md"))))
  (test-equal nil (probe-file (get-path "mapped.txt")))
  (test-file "keep.txt" (new-lines "keep")))
```

### Create directories and files

[Verify file creation for Chinese paths and paths with spaces.](tests/codoc-tests.lisp#L91-L99)  
```commonlisp
(defun test-creation ()
  (set-file "create.md"
            (new-lines "[中文](新目录/代码.lisp#L3-L4)" "```commonlisp"
                       ";; 你好。" "(print :hello)" "```"
                       "[Spaces](<space dir/code.lisp#L1>)" "~~~" "hello" "~~~"))
  (test-equal '(2 2)
              (multiple-value-list (codoc:set-document (get-path "create.md"))))
  (test-file "新目录/代码.lisp" (new-lines "" "" ";; 你好。" "(print :hello)"))
  (test-file "space dir/code.lisp" (new-lines "hello")))
```

## Markdown parsing tests

These tests cover ignored code blocks, fence boundaries, and line-ending formats.  

### Ignore unmapped code

[Verify that code blocks without a valid local purpose link do not generate files.](tests/codoc-tests.lisp#L101-L112)  
```commonlisp
(defun test-unmapped-blocks ()
  (set-file "ignored.md"
            (new-lines "```markdown" "[Example](never.lisp#L1)" "~~~"
                       "content" "~~~" "```"
                       "[Website](https://example.com/file#L1)" "```" "web" "```"
                       "[Normal link](plain.lisp)" "```" "plain" "```"
                       "[Stale link](stale.lisp#L1)" "Intervening prose."
                       "```" "stale" "```"))
  (test-equal '(0 0)
              (multiple-value-list (codoc:set-document (get-path "ignored.md"))))
  (dolist (name '("never.lisp" "plain.lisp" "stale.lisp"))
    (test-equal nil (probe-file (get-path name)))))
```

### Fences and indentation

[Verify that fence parsing keeps code content and its relative indent.](tests/codoc-tests.lisp#L114-L122)  
```commonlisp
(defun test-fences ()
  (set-file "fences.md"
            (new-lines "[Fence contents](fences.txt#L1-L4)" "  ````text"
                       "  first" "  ```" "  ~~~" "    indented" "  `````"))
  (codoc:set-document (get-path "fences.md"))
  (test-file "fences.txt" (new-lines "first" "```" "~~~" "  indented"))
  (set-file "blank.md" (new-lines "[Blank](blank.txt#L1-L2)" "```" "" "last" "```"))
  (codoc:set-document (get-path "blank.md"))
  (test-file "blank.txt" (new-lines "" "last")))
```

### Normalize line endings

[Verify that generated files use LF line endings and keep a trailing newline.](tests/codoc-tests.lisp#L124-L139)  
```commonlisp
(defun test-line-endings ()
  (set-file "ending.txt" (format nil "a~C~Cb~C~Cc" #\Return #\Newline #\Return #\Newline))
  (set-file "ending.md" (new-lines "[Replace](ending.txt#L2)" "```" "B" "```"))
  (codoc:set-document (get-path "ending.md"))
  (test-file "ending.txt" (new-lines "" "B"))
  (set-file "ending.md" (new-lines "[Extend](ending.txt#L5)" "```" "E" "```"))
  (codoc:set-document (get-path "ending.md"))
  (test-file "ending.txt" (new-lines "" "" "" "" "E"))
  (set-file "no-newline.txt" "old")
  (set-file "ending.md" (new-lines "[Last](no-newline.txt#L1)" "```" "new" "```"))
  (codoc:set-document (get-path "ending.md"))
  (test-file "no-newline.txt" (new-lines "new"))
  (set-file "crlf.md" (format nil "[CRLF](crlf.txt#L1)~C~C```~C~Cok~C~C```"
                              #\Return #\Newline #\Return #\Newline #\Return #\Newline))
  (codoc:set-document (get-path "crlf.md"))
  (test-file "crlf.txt" (new-lines "ok")))
```

## Validation and path tests

These tests check that invalid mappings are rejected before files are changed.  

### Reject invalid mappings

[Verify that range errors and fence errors do not change target files.](tests/codoc-tests.lisp#L141-L165)  
```commonlisp
(defun test-validation ()
  (set-file "protected.txt" (new-lines "original"))
  (let ((valid (new-lines "[Valid](protected.txt#L1)" "```" "changed" "```")))
    (test-rejection (concatenate 'string valid
                                (new-lines "[Mismatch](missing/file.txt#L1-L2)"
                                           "```" "one" "```"))
                    "expected 2 code lines, found 1")
    (test-file "protected.txt" (new-lines "original"))
    (test-equal nil (probe-file (get-path "missing/")))
    (test-rejection (concatenate 'string valid
                                (new-lines "[Overlap](./protected.txt#L1)"
                                           "```" "collision" "```"))
                    "overlaps")
    (test-file "protected.txt" (new-lines "original")))
  (dolist (range '("L0-L1" "L2-L1" "Lx-L2" "L1-L" "L1-L2-L3"))
    (test-rejection (new-lines (format nil "[Bad](bad.txt#~A)" range)
                               "```" "line" "```")
                    "invalid.md:1"))
  (test-rejection (new-lines "[Open](bad.txt#L1)" "```" "line") "no closing fence")
  (test-rejection (new-lines "[Empty](bad.txt#L1)" "```" "```") "found 0")
  (test-rejection (new-lines "[No path](#L1)" "```" "line" "```") "file is missing")
  (test-rejection (new-lines "[Alias](new/../alias.txt#L1)" "```" "one" "```"
                             "[Alias](alias.txt#L1)" "```" "two" "```")
                  "overlaps")
  (test-equal nil (probe-file (get-path "bad.txt"))))
```

### Handle path aliases

[Verify that target path checks recognize symlink aliases and directories.](tests/codoc-tests.lisp#L167-L197)  
```commonlisp
(defun test-target-paths ()
  (set-file "real/file.txt" (new-lines "original"))
  (test-equal 0
              (sb-ext:process-exit-code
               (sb-ext:run-program "/bin/ln"
                                   (list "-s" (namestring (get-path "real/"))
                                         (namestring (get-path "alias")))
                                   :wait t)))
  (test-rejection (new-lines "[Real](real/file.txt#L1)" "```" "first" "```"
                             "[Alias](alias/file.txt#L1)" "```" "second" "```")
                  "overlaps")
  (test-file "real/file.txt" (new-lines "original"))
  (test-rejection (new-lines "[New](real/new.txt#L1)" "```" "first" "```"
                             "[Alias](alias/new.txt#L1)" "```" "second" "```")
                  "overlaps")
  (test-equal nil (probe-file (get-path "real/new.txt")))
  (set-file "absolute.md"
            (new-lines (format nil "[Absolute](~A#L1)" (get-path "no-extension"))
                       "```" "absolute" "```"))
  (codoc:set-document (get-path "absolute.md"))
  (test-file "no-extension" (new-lines "absolute"))
  (test-rejection (new-lines "[Valid](real/file.txt#L1)" "```" "changed" "```"
                             "[Directory](real/#L1)" "```" "invalid" "```")
                  "Target must name a file")
  (test-file "real/file.txt" (new-lines "original"))
  (test-rejection (new-lines "[Valid](real/file.txt#L1)" "```" "changed" "```"
                             "[Directory](real#L1)" "```" "invalid" "```")
                  "Target must name a file")
  (test-file "real/file.txt" (new-lines "original"))
  ;; Remove the link explicitly before deleting the temporary directory tree.
  (delete-file (get-path "alias")))
```

## Command-line tests

These tests run the real binary and check exit codes, diagnostics, and default-document behavior.  

### Get a command result

[Return a test command's exit code and combined output.](tests/codoc-tests.lisp#L199-L204)  
```commonlisp
(defun get-command-result (&rest arguments)
  (let* ((output (make-string-output-stream))
         (process (sb-ext:run-program *executable* arguments
                                     :directory *root* :output output :error output
                                     :search nil :wait t)))
    (values (sb-ext:process-exit-code process) (get-output-stream-string output))))
```

### Arguments and error diagnostics

[Verify file generation, deletion, and error diagnostics for command arguments.](tests/codoc-tests.lisp#L206-L237)  
```commonlisp
(defun test-command-line ()
  (set-file "CODOC.md" (new-lines "[Default](default.txt#L1)" "```" "default" "```"))
  (test-equal 0 (get-command-result "get"))
  (test-file "default.txt" (new-lines "default"))
  (set-file "custom name.md" (new-lines "[Custom](custom.txt#L1)" "```" "custom" "```"))
  (test-equal 0 (get-command-result "get" "custom name.md"))
  (test-file "custom.txt" (new-lines "custom"))
  (multiple-value-bind (status output) (get-command-result "del" "custom name.md")
    (test-equal 0 status)
    (test-equal (new-lines "Deleted 1 mapped file.") output))
  (test-equal nil (probe-file (get-path "custom.txt")))
  (test-file "default.txt" (new-lines "default"))
  (test-equal 0 (get-command-result "del"))
  (test-equal nil (probe-file (get-path "default.txt")))
  (test-equal 0 (get-command-result "--help"))
  (test-equal 2 (get-command-result))
  (test-equal 2 (get-command-result "unknown"))
  (test-equal 2 (get-command-result "--unknown"))
  (test-equal 2 (get-command-result "get" "one.md" "two.md"))
  (test-equal 1 (get-command-result "get" "not-found.md"))
  (test-equal 1 (get-command-result "del" "not-found.md"))
  (set-file "cli-invalid.md" (new-lines "[Bad](bad.txt#L2-L1)" "```" "bad" "```"))
  (multiple-value-bind (status output) (get-command-result "get" "cli-invalid.md")
    (test-equal 1 status)
    (incf *checks*)
    (unless (and (search "cli-invalid.md:1" output) (search "reversed" output))
      (error "Missing command-line diagnostic: ~A" output)))
  (multiple-value-bind (status output) (get-command-result "del" "cli-invalid.md")
    (test-equal 1 status)
    (incf *checks*)
    (unless (search "reversed" output)
      (error "Missing command-line diagnostic: ~A" output))))
```

### Default document and example

[Verify default-document creation and regeneration of the greeting example.](tests/codoc-tests.lisp#L239-L269)  
```commonlisp
(defun test-default-document ()
  (let ((*root* (get-path "bootstrap/")))
    (ensure-directories-exist *root*)
    (test-equal 0 (get-command-result "--help"))
    (test-equal 2 (get-command-result "--unknown"))
    (test-equal 2 (get-command-result "get" "one.md" "two.md"))
    (test-equal 1 (get-command-result "get" "not-found.md"))
    (test-equal 1 (get-command-result "get" "CODOC.md"))
    (test-equal 1 (get-command-result "del"))
    (test-equal nil (probe-file (get-path "CODOC.md")))
    (test-equal nil (probe-file (get-path "examples/")))
    (multiple-value-bind (status output) (get-command-result "get")
      (test-equal 0 status)
      (test-equal (new-lines "Wrote 1 code block to 1 file.") output))
    (let ((document (codoc::get-file-text (get-path "CODOC.md")))
          (greeting (new-lines "(defun get-greeting ()" "  \"Hello from codoc.\")")))
      (test-file "examples/hello.lisp" greeting)
      (test-equal 0 (get-command-result "get"))
      (test-file "CODOC.md" document)
      (test-file "examples/hello.lisp" greeting)
      (set-file "examples/hello.lisp" (new-lines "old" "code" "trailing"))
      (test-equal 0 (get-command-result "get"))
      (test-file "CODOC.md" document)
      (test-file "examples/hello.lisp" greeting)
      (delete-file (get-path "CODOC.md"))
      (set-file "examples/hello.lisp" (new-lines "conflicting" "example" "trailing"))
      (set-file "examples/other.lisp" (new-lines "keep"))
      (test-equal 0 (get-command-result "get"))
      (test-file "CODOC.md" document)
      (test-file "examples/hello.lisp" greeting)
      (test-file "examples/other.lisp" (new-lines "keep")))))
```

## Test execution

The temporary directory is cleaned up after the tests finish or fail.  

### Run the full tests

[Run every test in a temporary directory and clean up the test files.](tests/codoc-tests.lisp#L271-L290)  
```commonlisp
(defun test-all ()
  (let ((*root* (merge-pathnames
                (format nil "codoc-tests-~D-~D/" (get-universal-time) (random 1000000000))
                (uiop:temporary-directory))))
    (ensure-directories-exist *root*)
    (unwind-protect
         (progn
           (test-ranges)
           (test-clean-targets)
           (test-del-document)
           (test-creation)
           (test-unmapped-blocks)
           (test-fences)
           (test-line-endings)
           (test-validation)
           (test-target-paths)
           (test-command-line)
           (test-default-document)
           (format t "Passed ~D checks.~%" *checks*))
      (uiop:delete-directory-tree *root* :validate t))))
```

### Start the tests

[Run the test suite.](tests/codoc-tests.lisp#L292)  
```commonlisp
(test-all)
```
