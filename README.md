# `clj-pdf` [![Build Status](https://travis-ci.org/yogthos/clj-pdf.svg?branch=master)](http://travis-ci.org/yogthos/clj-pdf) [![Coverage Status](https://coveralls.io/repos/yogthos/clj-pdf/badge.svg?branch=master)](https://coveralls.io/r/yogthos/clj-pdf?branch=master) [![Downloads](https://jarkeeper.com/yogthos/clj-pdf/downloads.svg)](https://jarkeeper.com/yogthos/clj-pdf)


A library for easily generating PDFs from Clojure. An example PDF is available [here](https://github.com/yogthos/clj-pdf/raw/master/example.pdf) with its source [below](#a-complete-example).

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

  - [Installation](#installation)
  - [Usage](#usage)
    - [Templating](#templating)
    - [Stylesheets](#stylesheets)
  - [Document Elements](#document-elements)
  - [Document Format](#document-format)
    - [Metadata](#metadata)
      - [Font](#font)
      - [Using Custom TTF Fonts](#using-custom-ttf-fonts)
    - [Document sections](#document-sections)
      - [Anchor](#anchor)
      - [Chapter](#chapter)
      - [Chunk](#chunk)
      - [Clear double page](#clear-double-page)
      - [Graphics](#graphics)
      - [Heading](#heading)
      - [Image](#image)
      - [Line](#line)
      - [List](#list)
      - [Pagebreak](#pagebreak)
      - [Paragraph](#paragraph)
      - [Phrase](#phrase)
      - [Reference](#reference)
      - [Section](#section)
      - [Spacer](#spacer)
      - [String](#string)
      - [Subscript](#subscript)
      - [Superscript](#superscript)
      - [SVG](#svg)
      - [Table](#table)
      - [PDF Table](#pdf-table)
      - [Table Cell](#table-cell)
      - [PDF Table Cell](#pdf-table-cell)
    - [Charting](#charting)
      - [bar chart](#bar-chart)
      - [line chart](#line-chart)
      - [pie chart](#pie-chart)
    - [A complete example](#a-complete-example)
- [Users](#users)
- [License](#license)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Installation

`clj-pdf` is available as a Maven artifact from [Clojars](https://clojars.org/search?q=clj-pdf):

Leiningen

[![Clojars Project](http://clojars.org/clj-pdf/latest-version.svg)](http://clojars.org/clj-pdf)

Maven

```xml
<repository>
  <id>clojars</id>
  <url>http://clojars.org/repo</url>
</repository>

<dependency>
  <groupId>clj-pdf</groupId>
  <artifactId>clj-pdf</artifactId>
  <version>2.2.1</version>
</dependency>
```

## Usage

PDF documents are generated calling the `pdf` function, defined in the `clj-pdf.core` namespace, with
input and output parameters.

`(pdf in out)`

`in` can be either a vector containing the document or an input stream. If `in` is an input stream then the forms will be read sequentially from it.

`out` can be either a string, in which case it's treated as a file name, or an output stream.

NOTE: using the `:pages` option will cause the complete document to reside in memory for post processing.


The documents contain a map with metadata followed by one or more elements. Each element must be a sequence starting with
a keyword specifying the element name or a string which will be treated as a paragraph.

Here's a basic example of a document:
```clojure
(ns example
  (:use clj-pdf.core))

(pdf
  [{}
   [:list {:roman true}
          [:chunk {:style :bold} "a bold item"]
          "another item"
          "yet another item"]
   [:phrase "some text"]
   [:phrase "some more text"]
   [:paragraph "yet more text"]]
  "doc.pdf")
```
and the resulting PDF output
<br/>
<img src="https://raw.github.com/yogthos/clj-pdf/master/example.png" hspace="20" alt="example"/>

Multiple documents can be combined into a single PDF using the `clj-pdf.core/collate` function.
The function accepts an output stream followed by two or more documents. The documents can be one
of InputStream, file name, URL, or a byte array.

```clojure
(def doc1 (java.io.ByteArrayOutputStream.))
(def doc2 (java.io.ByteArrayOutputStream.))
(def doc3 (java.io.ByteArrayOutputStream.))

(pdf [{} "first document"] doc1)
(pdf [{} "second document"] doc2)
(pdf [{} "third document"] doc2)

(collate (java.io.FileOutputStream. (clojure.java.io/file "merged.pdf"))
         (.toByteArray doc1)
         (.toByteArray doc1)
         (.toByteArray doc1))
```

Sequences containing elements will be expanded into the document:

```clojure
(pdf
 [{}
   (for [i (range 3)]
    [:paragraph (str "item: " i)])]
 "doc.pdf")
```
is equivalent to
```clojure
(pdf
 [{}
  [:paragraph "item: 0"]
  [:paragraph "item: 1"]
  [:paragraph "item: 2"]]
 "doc.pdf")
```

Since `clj-pdf` uses regular Clojure vectors you can easily add your own helper functions as well.
For example, a `pdf-table` is expected to have the following format:

```clojure
[:pdf-table
  [10 20 15]
  ["foo" [:chunk {:style :bold} "bar"] [:phrase "baz"]]
  [[:pdf-cell "foo"] [:pdf-cell "foo"] [:pdf-cell "foo"]]
  [[:pdf-cell "foo"] [:pdf-cell "foo"] [:pdf-cell "foo"]]]
```
We can add a helper generate the expected format from the given data:

```clojure
(defn pdf-table [column-widths & rows]
  (into
    [:pdf-table column-widths]
    (map (partial map (fn [element] [:pdf-cell element])) rows)))

(pdf-table
  [10 20 15]
  ["foo" [:chunk {:style :bold} "bar"] [:phrase "baz"]]
  ["foo" "foo" "foo"]
  ["foo" "foo" "foo"])
```

### Templating

The library provides some rudimentary templating options, the `template` macro can be used to generate a function which accepts a sequence of maps,
and applies the template to each item. This is primarily meant to complement working with [clojure.java.jdbc](https://github.com/clojure/java.jdbc/),
which returns sequences of maps representing the table rows.

The $ is used to indicate the anchors in the template. These will be swapped with the values from the map with
the corresponding keys. For example, given a vector of maps, such as:

```clojure
(def employees
  [{:country "Germany",
    :place "Nuremberg",
    :occupation "Engineer",
    :name "Neil Chetty"}
   {:country "Germany",
    :place "Ulm",
    :occupation "Engineer",
    :name "Vera Ellison"}])
```
and a template
```clojure
(def employee-template
  (template
    [:paragraph
     [:heading $name]
     [:chunk {:style :bold} "occupation: "] $occupation "\n"
     [:chunk {:style :bold} "place: "] $place "\n"
     [:chunk {:style :bold} "country: "] $country
     [:spacer]]))
```
the following output will be produced when the template is applied to the data:
```clojure
(employee-template employees)

=>

'([:paragraph [:heading "Neil Chetty"]
   [:chunk {:style :bold} "occupation: "] "Engineer" "\n"
   [:chunk {:style :bold} "place: "] "Nuremberg" "\n"
   [:chunk {:style :bold} "country: "] "Germany" [:spacer]]
  [:paragraph [:heading "Vera Ellison"]
   [:chunk {:style :bold} "occupation: "] "Engineer" "\n"
   [:chunk {:style :bold} "place: "] "Ulm" "\n"
   [:chunk {:style :bold} "country: "] "Germany" [:spacer]])
```

It is also possible to apply post processing to the anchors in the template:
```clojure
(def employee-template-paragraph
  (template
    [:paragraph
     [:heading (if (and $name (.startsWith $name "Alfred"))
                 (.toUpperCase $name) $name)]
     [:chunk {:style :bold} "occupation: "] $occupation "\n"
     [:chunk {:style :bold} "place: "] $place "\n"
     [:chunk {:style :bold} "country: "] $country
     [:spacer]]))
```

### Stylesheets

To use a CSS-like stylesheet, you can create a stylesheet map of class names.
Each class name will have an associated attribute map.  A specific stylesheet
must be included in the document metadata map in order to use it.

Use the CSS-like shortcut for applying classes to elements (e.g. `[:paragraph.foo.bar]`):

```clojure
(def stylesheet
  {:foo {:color [255 0 0]
         :family :helvetica}
   :bar {:color [0 0 255]
         :family :helvetica}
   :baz {:align :right}})

(pdf
 [{:stylesheet stylesheet}
  [:paragraph.foo "item: 0"]
  [:paragraph.bar "item: 1"]
  [:paragraph.bar.baz "item: 2"]]
 "doc.pdf")
```

## Document Elements

[Anchor](#anchor),
[Chapter](#chapter),
[Chart](#charting),
[Chunk](#chunk),
[Clear double page](#clear-double-page),
[Graphics](#graphics),
[Heading](#heading),
[Image](#image),
[Line](#line),
[List](#list),
[Pagebreak](#pagebreak),
[Paragraph](#paragraph),
[Phrase](#phrase),
[Reference](#reference),
[Section](#section),
[Spacer](#spacer),
[String](#string),
[Subscript](#subscript),
[Superscript](#superscript),
[SVG](#svg),
[Table](#table),
[Table Cell](#table-cell)

## Document Format

### Metadata

All fields in the metadata section are optional:

```clojure
{:title  "Test doc"
 :left-margin   10
 :right-margin  10
 :top-margin    20
 :bottom-margin 25
 :subject "Some subject"
 :size          :a4
 :orientation   :landscape
 :author "John Doe"
 :creator "Jane Doe"
 :font  {:size 11} ;specifies default font
 :doc-header ["inspired by" "William Shakespeare"]

 ;;a watermark can be specified as an image or a function that
 ;;takes the Graphics2D context and render an image on it
 ;;the watermark will be automatically applied to each page in the document
 ;;optionally the watermark can be rotated, scaled, and translated
 :watermark
 {:image "watermark.jpg"
  ;; :image and :render keys are exclusive, :render is preferred
  :render (fn [g2d] (.drawString g2d "DRAFT COPY" 0 0))
  :translate [100 200]
  :rotate 45
  :scale 8}

 :header "Page header text appears on each page"
 :letterhead ["A simple Letter head"] ;Sequence of any elements. If set, the first page shows letterhead instead of header

 ;;setting :footer to false will pevent page numbers from being displayed
 :footer {:text "Page footer text appears on each page (includes page number)"
          :align :left ;optional footer alignment of :left|:right|:center defaults to :right
          :footer-separator "text which will be displayed between current page number and total pages, defaults to /"
          :start-page 2 ;optional parameter to indicate on what page the footer starts, has no effect when :pages is set to false
          :page-numbers false ;should page numbers be printed in the footer, defaults to true
         }


 ;; specifies if total pages should be printed in the footer of each page
 :pages true

 ;; references can be used to cache compiled items for faster compilation,
 ;; see the :reference tag for details
 :references {:batman [:image "batman.jpg"]
              :superman [:image "superman.png"]}

 ;; Register TTF fonts in some probable directories, set this to true if
 ;; you're going to use :ttf-name to set custom system fonts
 :register-system-fonts? true
}
```

The `:header` and `:footer` keys can also point to a `:table` element. The `:table` key
must point to a `:pdf-table` type element:

```clojure
{:header {:x 20
          :y 50
          :table
          [:pdf-table
          {:border false}
          [20 15 60]
          ["This is a table header" "second column" "third column"]]}
 :footer {:table {:table
          [:pdf-table
          {:border false}
          [20 15 60]
          ["This is a table footer" "second column" "third column"]]}}
```

The `:x` and `:y` keys can be used on the header and footer when using the `:table` key to specify the x/y offset on the page explicitly.

available page sizes:

```clojure
:a0
:a1
:a10
:a2
:a3
:a4
:a5
:a6
:a7
:a8
:a9
:arch-a
:arch-b
:arch-c
:arch-d
:arch-e
:b0
:b1
:b10
:b2
:b3
:b4
:b5
:b6
:b7
:b8
:b9
:crown-octavo
:crown-quarto
:demy-octavo
:demy-quarto
:executive
:flsa
:flse
:halfletter
:id-1
:id-2
:id-3
:large-crown-octavo
:large-crown-quarto
:ledger
:legal
:letter
:note
:penguin-large-paperback
:penguin-small-paperback
:postcard
:royal-octavo
:royal-quarto
:small-paperback
:tabloid
```

Alternatively, explicit page size can also be specified using a vector, eg:

```clojure
:size [1296 1296]
```
The size defaults to A4 page size if none is provided.

Orientation defaults to portrait, unless `:landscape` is specified.

#### Font

A font is defined by a map consisting of the following parameters, all parameters are optional

* :family has following options: :courier, :helvetica, :times-roman, :symbol, :zapfdingbats defaults to :helvetica
* :ttf-name is the name of a TTF font installed on the system. Overrides :family parameter. It could be an absolute or relative path to a font file; it will also seek for a font in classpath resources.
* :encoding should be set to :unicode to enable unicode support if custom :ttf-name font is used
* :size is a number default is 10
* :style has following options: :bold, :italic, :bold-italic, :normal, :strikethru, :underline defaults to :normal
* :styles a vector of multiple style keys
* :color is a vector of [r g b] defaults to black

example font:

```clojure
{:style :bold
 :size 18
 :family :helvetica
 :color [0 234 123]}

 {:styles [:bold :underline]
  :family :helvetica}
```
note: Font styles are additive, for example setting style :italic on the phrase, and then size 20 on a chunk inside the phrase, will result with the chunk having italic font of size 20. Inner elements can override style set by their parents.

#### Using Custom TTF Fonts

For non-ASCII text output you will probably have to use external font and define `:encoding` as `:unicode`.

The following example illustrates how to specify a custom font file such as Cyrillic font.

```clojure
(pdf
  [{:font {:encoding :unicode
           :ttf-name "fonts/Arialuni.ttf"}}
  [:phrase "тест 123"]]
  "doc.pdf")
```

Custom fonts can also be specified for any elements that support font metadata, such as phrases and paragraphs:

```clojure
[:paragraph
   {:encoding "UniJIS-UCS2-H"
    :ttf-name "HeiseiKakuGo-W5"}
   "こんにちは世界"]
```

You could set `:ttf-name` as absolute or relative path to the font file. It will also load fonts from classpath resources by default.

### Document sections

Each document section is represented by a vector starting with a keyword identifying the section followed by an optional map of metadata and the contents of the section.

#### Anchor

tag :anchor

optional metadata:

* :id name of the anchor
* :target an external link or a name of the anchor this anchor points to, if referencing another anchor then prefix target with #
* :style font
* :styles a vector of font styles
* :leading number

content:

idiosynchorsies:

* when both font style and leading number are specified the content must be a string
* when leading number is specified content can be a chunk or a string
* when only font style is specified content must be a string
* if no font style or leading is specified then content can be a chunk, a phrase, or a string

```clojure
[:anchor {:target "http://google.com"} "google"]

[:anchor {:style {:size 15} :leading 20 :id "targetAnchor"} "some anchor"]

[:anchor {:target "#targetAnchor"} "this anchor points to some anchor"]

[:anchor [:phrase {:style :bold} "some anchor phrase"]]

[:anchor "plain anchor"]
```

#### Chapter

tag :chapter

optional metadata:

* none

content:

* string
* paragraph

```clojure
[:chapter "First Chapter"]

[:chapter [:paragraph "Second Chapter"]]
```

#### Chunk

tag :chunk

optional metadata:

* :sub boolean sets chunk to subscript
* :super boolean sets chunk to superscript

font metadata (refer to Font section for details)

* :family
* :ttf-name
* :size
* :style
* :styles
* :color
* :background [r b g]

Note that when using `:ttf-name`, you should set `:register-system-fonts? true` in the document metadata in order to load the available system fonts, or manually provide paths to the font files.

```clojure
[:chunk {:style :bold} "small chunk of text"]

[:chunk {:styles [:bold :italic]} "small chunk of text"]

[:chunk {:background [0 255 0]} "green chunk"]

[:chunk {:color [0 0 0] :background [255 0 0]} "more fun with color"]

[:chunk {:super true} "5"]

[:chunk {:sub true} "2"]
```

#### Clear double page

tag :clear-double-page

Ends current page and inserts a blank page if necessary to ensure that subsequent content starts on the next odd-numbered page. In other words, if you print the resulting PDF on double-sided paper, the content that comes after a `:clear-double-page` will always be on a different sheet of paper from the content that came before it.

```clojure
;; Example documents

[[:paragraph "this is on page 1"] [:clear-double-page] [:paragraph "this is on page 3"]]

[[:paragraph "this is on page 1"]
 [:clear-double-page] [:clear-double-page]
 [:paragraph "this is on page 3"]]

[[:paragraph "this is on page 1"] [:pagebreak]
 [:paragraph "this is on page 2"] [:clear-double-page]
 [:paragraph "this is on page 3"]]

[[:paragraph "this is on page 1"] [:pagebreak]
 [:paragraph "this is on page 2"] [:pagebreak]
 [:paragraph "this is on page 3"] [:clear-double-page]
 [:paragraph "this is on page 5"]]

;; :clear-double-page on an empty page 1 does nothing
[[:clear-double-page] [:paragraph "This is on page 1"]]
```

#### Graphics

tag :graphics

The command takes a function with a single argument, the Graphics2D object, onto which you can draw things. Note that this is actually the *PdfGraphics2D* object which
will render the drawing instructions as vectors rather than to a raster bitmap. There is no need to dispose of the graphics context as this is done on exiting the function.
The co-ordinates are absolute from the top left hand side of the current page. There are no restrictions as to the number of times this command can be invoked per page; subsequent
graphics drawings will be overlaid on prior renderings.

optional metadata:

* :under boolean when true, the rendered graphics are drawn under the page (useful for watermarking), else defaults to above the page
* :translate ```[dx dy]``` shifts the graphic rendering by (dx,dy)
* :scale ```[sx sy]``` or ```s``` scales the graphic rendering by (sx,sy), or (s,s)
* :rotate ```radians``` rotates the graphic rendering by the given angle (in radians)

```clojure
[:graphics {:under true :translate [100 100]}
 (fn [g2d]
   (doto g2d
     (.setColor java.awt.Color/RED)
     (.drawOval (int 0) (int 0) (int 50) (int 50))))]
```

#### Heading

tag :heading

optional metadata:

* :align specifies alignement of heading possible valuse :left, :center, :right, :justified
* :style font (refer to Font section for details)
* :styles font (refer to Font section for details)

```clojure
[:heading "Lorem Ipsum"]

[:heading {:style {:size 15}} "Lorem Ipsum"]

[:heading {:style {:size 10 :color [100 40 150] :align :right}} "Foo"]
```

#### Image

tag :image

image data can be one of java.net.URL, java.awt.Image, byte array, base64 string, or a string representing URL or a file,
images larger than the page margins will automatically be scaled to fit.

optional metadata:

* :scale  number - percentage relative to original image size
* :xscale number - percentage relative to page size
* :yscale num - percentage relative to page size
* :width num - set width for image: overrides scaling
* :height num - set height for image: overrides scaling
* :align :left, :center, :right, :justified
* :annotation ["title" "text"]
* :pad-left number
* :pad-right number
* :base64 boolean - if set the image is expected to be a Base64 string

```clojure

[:image
   {:xscale     0.5
    :yscale     0.8
    :align      :center
    :annotation ["FOO" "BAR"]
    :pad-left   100
    :pad-right  50}
   (javax.imageio.ImageIO/read (-> "mandelbrot.jpg" clojure.java.io/resource clojure.java.io/file) )]
[:image "test/mandelbrot.jpg"]
[:image "http://clojure.org/space/showimage/clojure-icon.gif"]

; images can also be inserted inline with other text by wrapping it inside
; of a chunk element
[:paragraph "hello, world!" [:chunk [:image "smiley.png"]]]

; x and y values provided to the chunk are relative offsets for the image.
; the image element itself still accepts it's normal properties shown above
[:chunk {:x 10 :y 10} [:image {:width 16 :height 16} "smiley.png"]]
```

#### Line

tag :line

optional metadata:

* :dotted boolean
* :gap number spaces between dots if line is dotted

creates a horizontal line

```clojure
[:line]
[:line {:dotted true}]
[:line {:dotted true :gap 10}]
```


#### List

tag :list

optional metadata:

* :numbered            boolean
* :lettered            boolean
* :roman               boolean
* :greek               boolean
* :dingbats            boolean
* :dingbats-char-num   boolean
* :dingbatsnumber      boolean
* :dingbatsnumber-type boolean
* :lowercase           boolean
* :indent              number
* :symbol              string (specifies the symbol to use for list items, defaults to "-")


content:

* strings, phrases, or chunks


```clojure
[:list {:roman true}
       [:chunk {:style :bold} "a bold item"]
       "another item"
       "yet another item"]
[:list {:symbol "*"}
       [:chunk {:style :bold} "a bold item"]
       "another item"
       "yet another item"]
```

#### Pagebreak

tag :pagebreak

Creates a new page in the document, subsequent content will start on that page.
Only creates a new page if the current page is not blank; otherwise, it's ignored.

```clojure
[:pagebreak]
```

#### Paragraph

tag :paragraph

optional metadata:

* :indent            number (indentation for the paragraph)
* :first-line-indent number (indentation for the first line of the paragraph)
* :keep-together     boolean
* :leading           number (line spacing is measured in 72 units per inch, default spacing is 1.5 times the font height)
* :align             :left, :center, :right, :justified

font metadata (refer to Font section for details)

* :family
* :ttf-name
* :size
* :style
* :styles
* :color

content:

* one or more elements (string, chunk, phrase, paragraph)

```clojure
[:paragraph "a fine paragraph"]

[:paragraph {:keep-together true :indent 20} "a fine paragraph"]

[:paragraph
  {:style :bold :size 10 :family :helvetica :color [0 255 221]}
  "Lorem ipsum dolor sit amet, consectetur adipiscing elit."]

; font set in the paragraph can be modified by its children:
[:paragraph {:indent 50 :color [0 255 221]}
  [:phrase {:style :bold :size 18 :family :helvetica} "Hello Clojure!"]]

[:paragraph "256" [:chunk {:super true} "5"] " or 128" [:chunk {:sub true} "2"]]

```

#### Phrase

tag :phrase

optional metadata:

* :leading number (line spacing is measured in 72 units per inch, default spacing is 1.5 times the font height)

font metadata (refer to Font section for details)

* :family
* :ttf-name
* :size
* :style
* :styles
* :color

content:

* strings and chunks


```clojure
[:phrase "some text here"]

[:phrase {:style :bold :size 18 :family :helvetica :color [0 255 221]}
         "Hello Clojure!"]

[:phrase [:chunk {:style :italic} "chunk one"]
         [:chunk {:size 20} "Big text"]
         "some other text"]
```

#### Reference

tag :reference

A reference tag can be used to cache repeating items. The references must be defined in the document metadata section. Both `:image` and `:chart` tags are cached by default.

```clojure
[:reference :reference-id]

(time
  (pdf [{:references {:repeating [:paragraph "I repeat a lot!"]}}
        (for [i (range 10000)]
          [:reference :repeating])]
     "super.pdf"))
"Elapsed time: 165.483 msecs"

(time
  (pdf [{}
        (for [i (range 10000)]
          [:paragraph "I repeat a lot!"])]
     "super.pdf"))
"Elapsed time: 584.544 msecs"
```

#### Section

tag :section

Chapter has to be the root element for any sections. Subsequently sections can only be parented under chapters and other sections, a section must contain a title followed by the content

optional metadata:

* :indent number

```clojure
[:chapter [:paragraph {:color [250 0 0]} "Chapter"]
   [:section "Section Title" "Some content"]
   [:section [:paragraph {:size 10} "Section Title"]
    [:paragraph "Some content"]
    [:paragraph "Some more content"]
    [:section {:color [100 200 50]} [:paragraph "Nested Section Title"]
              [:paragraph "nested section content"]]]]
```

#### Spacer

tag :spacer

creates a number of new lines equal to the number passed in (1 space is default)

```clojure
[:spacer ] ;creates 1 new lines
[:spacer 5] ;creates 5 new lines
```

#### String

A string will be automatically converted to a paragraph

```
"this text will be treated as a paragraph"
```

#### Subscript

tag :subscript

optional metadata:

* :style font
* :styles fonts

creates a text chunk in subscript

```clojure
[:subscript "some subscript text"]

[:subscript {:style :bold} "some bold subscript text"]
```

#### Superscript

tag :superscript

optional metadata:

* :style font
* :styles fonts

creates a text chunk in subscript

```clojure
[:superscript "some superscript text"]

[:superscript {:style :bold} "some bold superscript text"]
```

#### SVG

tag :svg

Renders a string of text as an SVG document - use of [Hiccup](http://weavejester.github.io/hiccup/) is recommended here, or if a reader or file is presented, content is retrieved from
that resource.

optional metadata (refer to Graphics section for details):

* :under
* :translate
* :scale
* :rotate

```clojure
[:svg {}
  "<?xml version=\"1.0\" encoding=\"UTF-8\"?>
   <!DOCTYPE svg>
   <svg xmlns=\"http://www.w3.org/2000/svg\" width=\"304\" height=\"290\">
     <path d=\"M2,111 h300 l-242.7,176.3 92.7,-285.3 92.7,285.3 z\"
           style=\"fill:#FB2;stroke:#BBB;stroke-width:15;stroke-linejoin:round\"/>
   </svg>"]
```

```clojure
[:svg {} (clojure.java.io/file "pentagram.svg")]
```


#### Table

tag :table

metadata:

* :align table alignment on the page can be: :left, :center, :right, :justified
* :background-color  `[r g b]` (int values)
* :header `[{:backdrop-color [r g b]} "column name" ...]` if only a single column name is provided it will span all rows.
* :header can also be formatted via a collection of phrases or paragraphs `[{:backdrop-color [r g b]} [:paragraph ...]`
* :spacing number
* :padding number
* :border boolean
* :border-width number
* :cell-border boolean
* :width number signifying the percentage of the page width that the table will take up
* :widths vector list of column widths in percentage
* :header is a vector of strings, which specify the headers for each column, can optionally start with metadata for setting header color
   and can also be formatted via a vector of paragraphs or phrases
* :offset number
* :num-cols number

```clojure
[:table {:header ["Row 1" "Row 2" "Row 3"] :width 50 :border false :cell-border false}
  [[:cell {:colspan 2} "Foo"] "Bar"]
  [[:cell "foo1" " " "foo2"] "bar1" "baz1"]
  ["foo2" "bar2" "baz2"]]
  
[:table
  {:header ["A" "B" [:cell {:colspan 2 :align :center} "Cell"]]}
  ["1a" "1b" "1c" "1d"]
  ["2a" "2b" "2c" "2d"]
  ["3a" "3b" "3c" "3d"]
  ["4a" "4b" "4c" "4d"]]

;;header elements can set alignment
[:table {:header [{:backdrop-color [100 100 100]}
                  [:paragraph {:style :bold :size 15} "Foo"]
                  [:paragraph {:align :center :style :bold :size 15} "Bar"]]}
  ["foo" "bar"]]

[:table {:border-width 10 :header ["Row 1" "Row 2" "Row 3"]}
  ["foo" "bar" "baz"]
  ["foo1" "bar1" "baz1"]
  ["foo2" "bar2" "baz2"]]

[:table {:header [{:backdrop-color [100 100 100]}
                  [:paragraph {:style :bold :size 15} "FOO"]
                  [:paragraph {:size 20} "BAR"]]
         :spacing 20}
  ["foo" "bar"]]

;; the widths will be: a width of 50% for the first column,
;; 25% for the second and third column.
[:table {:border false
	     :widths [2 1 1]
         :header [{:backdrop-color [100 100 100]} "Singe Header"]}
  ["foo" "bar" "baz"]
  ["foo1" "bar1" "baz1"]
  ["foo2" "bar2" "baz2"]]

[:table {:cell-border false
         :header [{:backdrop-color [100 100 100]} "Row 1" "Row 2" "Row 3"]
         :spacing 20}
  ["foo"
    [:cell
      [:phrase {:style :italic :size 18 :family :helvetica :color [200 55 221]}
        "Hello Clojure!"]]
    "baz"]
  ["foo1" [:cell {:color [100 10 200]} "bar1"] "baz1"]
  ["foo2" "bar2" "baz2"]]
```

#### PDF Table

tag :pdf-table

pdf-table accepts metadata, followed by a vector specifying the width for each column, followed by columns
that can either be strings, images, chunks, paragraphs, phrases, pdf-cells, or other pdf-tables

metadata:

* :background-color `[r g b]`
* :spacing-before number
* :spacing-after number
* :cell-border boolean
* :bounding-box `[width height]`
* :horizontal-align :left, :rigth, :center, :justified
* :title string
* :width number
* :width-percent number (0-100)

```clojure
[:pdf-table
  {:bounding-box [50 100]
   :horizontal-align :right
   :spacing-before 100}
  [10 20 15]
  ["foo" [:chunk {:style :bold} "bar"] [:phrase "baz"]]
  [[:pdf-cell "foo"] [:pdf-cell "foo"] [:pdf-cell "foo"]]
  [[:pdf-cell "foo"] [:pdf-cell "foo"] [:pdf-cell "foo"]]]
```

#### Table Cell

Cells can be optionally used inside tables to provide specific style for table elements

tag :cell

metadata:

* :align :left, :center, :right, :justified
* :background-color `[r g b]` (int values)
* :colspan number
* :border boolean
* :set-border `[:top :bottom :left :right]` list of enabled borders, pass empty vector to disable all borders
* :border-width number
* :border-width-bottom number
* :border-width-left number
* :border-width-right number
* :border-width-top number

content:

Cell can contain any elements such as anchor, annotation, chunk, paragraph, or a phrase, which can each have their own style

note: Cells can contain other elements including tables

```clojure
[:cell {:colspan 2} "Foo"]

[:cell {:colspan 3 :rowspan 2} "Foo"]

[:cell [:phrase {:style :italic :size 18 :family :helvetica :color [200 55 221]} "Hello Clojure!"]]

[:cell {:color [100 10 200]} "bar1"]

[:cell [:table ["Inner table Col1" "Inner table Col2" "Inner table Col3"]]]
```

#### PDF Table Cell

PDF Table cells must be used inside PDF Tables

tag :pdf-cell

optional metadata:

* :background-color `[r g b]`
* :align :left, :center, :right, :justified
* :valign :top, :middle, :bottom
* :colspan number
* :rowspan number
* :border boolean
* :set-border `[:top :bottom :left :right]` list of enabled borders, pass empty vector to disable all borders
* :border-color `[r g b]`
* :border-width number
* :border-width-bottom number
* :border-width-left number
* :border-width-right number
* :border-width-top number
* :padding number (or a [CSS-like](https://developer.mozilla.org/en-US/docs/Web/CSS/padding#Examples) vector of numbers)
* :padding-bottom number
* :padding-left number
* :padding-right number
* :padding-top number
* :rotation number - rotates the cell
* :height - number
* :min-height - number

```clojure
[:pdf-cell {:colspan 2 :align :left} "Foo"]

[:pdf-table
  [10 20 15]
  [[:pdf-cell "foo"] [:pdf-cell "foo"] [:pdf-cell "foo"]]
  [[:pdf-cell {:min-height 40 :align :center :valign :middle} "foo"]
   [:pdf-cell {:valign :top} "foo"]
   [:pdf-cell {:valign :bottom} "foo"]]]
```

### Charting

tag :chart

metadata:

* :type        - bar-chart, line-chart, pie-chart
* :x-label     - only used for line and bar charts
* :y-label     - only used for line and bar charts
* :background  - a vector of [r g b] integer values
* :time-series - only used in line chart
* :time-format - can optionally be used with time-series to provide custom date formatting, defaults to "yyyy-MM-dd-HH:mm:ss"
* :horizontal  - can be used with bar charts and line charts, not supported by time series
* :title       - the title of the chart

additional image metadata (draws the chart as a raster bitmap image, default unless :vector is specified)

* :xscale number - percentage relative to page size
* :yscale num - percentage relative to page size
* :width num - set width for image: overrides scaling
* :height num - set height for image: overrides scaling
* :align "left|center|right|justified"
* :annotation ["title" "text"]
* :pad-left number
* :pad-right number

alternative vector metadata (used instead of the default image metadata, draws the chart as a scalable vector diagram)

* :vector boolean - when true, draws the chart at (0,0) on the page, unless the :translate argument is also supplied, in which case the drawing is offset accordingly.
* :width num - set width for chart
* :height num - set height for chart

optional vector metadata (refer to Graphics section for details):

* :under
* :translate
* :scale
* :rotate

#### bar chart

```clojure
[:chart
  {:type "bar-chart"
   :title "Bar Chart"
   :background [10 100 40]
   :x-label "Items"
   :y-label "Quality"}
  [2 "Foo"] [4 "Bar"] [10 "Baz"]]
```

The same chart rendered with vector drawing:

```clojure
[:chart {:type "bar-chart" :title "Bar Chart" :x-label "Items" :y-label "Quality"
         :vector true :width 500 :height 400 :translate [50 50]}
  [2 "Foo"] [4 "Bar"] [10 "Baz"]]
```

#### line chart

* :point-labels - boolean (show a label with the x,y position for each point)
* :show-points - boolean (display a box for each point in the data set)
* :label-percision - int (max number of digits after the decimal point)
* :label-format - string in format of "{0}:{1}:{2}" where {0} is the name of the series, {1} is the x value, and {2} is the y value
* :tick-interval - the range between ticks on each axis
* :tick-interval-x - the range between ticks on the x axis
* :tick-interval-y - the range between ticks on the y axis
* :range-x - a vector representing the start and end of the x axis range for the chart
* :range-y - a vector representing the start and end of the y axis range for the chart
* :time-format - string representing the date format in a time-series

if :time-series is set to true then items on x axis must be dates, the default format is "yyyy-MM-dd-HH:mm:ss", for custom formatting options refer [here](http://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html)

```clojure
[:chart {:type :line-chart :title "Line Chart" :x-label "checkpoints" :y-label "units"}
  ["Foo" [1 10] [2 13] [3 120] [4 455] [5 300] [6 600]]
  ["Bar" [1 13] [2 33] [3 320] [4 155] [5 200] [6 300]]]
```

```clojure
[:chart
  {:type :line-chart
   :x-label "foo"
   :y-label "bar"
   :tick-interval 1.5
   :range [10 30]}
  ["sample data set" [10 10] [20 11] [25 12] [30 15] [55 30]]]
```

```clojure
[:chart,
  {:x-label "time"
   :y-label "progress"
   :time-series true
   :title   "Time Chart"
   :type    :line-chart}
  ["Incidents"
   ["2011-01-03-11:20:11" 200]
   ["2011-02-11-22:25:01" 400]
   ["2011-04-02-09:35:10" 350]
   ["2011-07-06-12:20:07" 600]]]
```

```clojure
[:chart {:type :line-chart
         :time-series true
         :time-format "MM/yy"
         :title "Time Chart"
         :x-label "time"
         :y-label "progress"}
  ["Occurances" ["01/11" 200] ["02/12" 400] ["05/12" 350] ["11/13" 600]]]
```

#### pie chart

```clojure
[:chart {:type :pie-chart :title "Big Pie"} ["One" 21] ["Two" 23] ["Three" 345]]
```

### A complete example

Given these macros:

```clojure
(defn radians [degrees] (Math/toRadians degrees))

(defmacro rot [g2d angle & body]
  `(do (. ~g2d rotate (radians ~angle))
    (do ~@body)
    (. ~g2d rotate (radians (- 0 ~angle)))))

(defmacro trans [g2d dx dy & body]
  `(do (. ~g2d translate ~dx ~dy)
    (do ~@body)
    (. ~g2d translate (- 0 ~dx) (- 0 ~dy))))
```

creating a pdf:

```clojure
(import [java.awt Color])

(defn draw-tree [g2d length depth]
  (when (pos? depth)
    (.drawLine g2d 0 0 length 0)
    (trans g2d (int length) 0
      (rot g2d -30 (draw-tree g2d (* length 0.75) (- depth 1)))
      (rot g2d 30 (draw-tree g2d (* length 0.75) (- depth 1))))))

(pdf
  [{:title         "Test doc"
    :header        "page header"
    :subject       "Some subject"
    :creator       "Jane Doe"
    :doc-header    ["inspired by" "William Shakespeare"]
    :right-margin  50
    :author        "John Doe"
    :bottom-margin 25
    :left-margin   10
    :top-margin    20
    :size          "a4"
    :footer        "page"}

   [:heading "Lorem Ipsum"]

   [:paragraph
    "Lorem ipsum dolor sit amet, consectetur adipiscing elit. Donec non iaculis lectus. Integer vel libero libero. Phasellus metus augue, consequat a viverra vel, fermentum convallis sem. Etiam venenatis laoreet quam, et adipiscing mi lobortis sit amet. Fusce eu velit vitae dolor vulputate imperdiet. Suspendisse dui risus, mollis ut tempor sed, dapibus a leo. Aenean nisi augue, placerat a cursus eu, convallis viverra urna. Nunc iaculis pharetra pretium. Suspendisse sit amet erat nisl, quis lacinia dolor. Integer mollis placerat metus in adipiscing. Fusce tincidunt sapien in quam vehicula tincidunt. Integer id ligula ante, interdum sodales enim. Suspendisse quis erat ut augue porta laoreet."]

   [:paragraph
    "Sed pellentesque lacus vel sapien facilisis vehicula. Quisque non lectus lacus, at varius nibh. Integer porttitor porttitor gravida. Lorem ipsum dolor sit amet, consectetur adipiscing elit. Phasellus accumsan ante tincidunt magna dictum vulputate. Maecenas suscipit commodo leo sed mattis. Morbi dictum molestie justo eu egestas. Praesent lacus est, euismod vitae consequat non, accumsan in justo. Nam rhoncus dapibus nunc vel dignissim."]

   [:paragraph
    "Nulla id neque ac felis tempor pretium adipiscing ac tortor. Aenean ac metus sapien, at laoreet quam. Vivamus id dui eget neque mattis accumsan. Aliquam aliquam lacinia lorem ut dapibus. Fusce aliquam augue non libero viverra ut porta nisl mollis. Mauris in justo in nibh fermentum dapibus at ut erat. Maecenas vitae fermentum lectus. Nunc dolor nisl, commodo a pellentesque non, tincidunt id dolor. Nulla tellus neque, consectetur in scelerisque vitae, cursus vel urna. Phasellus ullamcorper ultrices nisi ac feugiat."]

   [:table {:header [{:background-color [100 100 100]} "FOO"] :spacing 2}
    ["foo"
     [:cell
      [:phrase
       {:style "italic" :size 18 :family "helvetica" :color [200 55 221]}
       "Hello Clojure!"]]
     "baz"]
    ["foo1" [:cell {:background-color [100 10 200]} "bar1"] "baz1"]
    ["foo2" "bar2" [:cell ["table" ["Inner table Col1" "Inner table Col2" "Inner table Col3"]]]]]

   [:paragraph
    "Suspendisse consequat, mauris vel feugiat suscipit, turpis metus semper metus, et vulputate sem nisi a dolor. Duis egestas luctus elit eget dignissim. Vivamus elit elit, blandit id volutpat semper, luctus id eros. Duis scelerisque aliquam lorem, sed venenatis leo molestie ac. Vivamus diam arcu, sodales at molestie nec, pulvinar posuere est. Morbi a velit ante. Nulla odio leo, volutpat vitae eleifend nec, luctus ac risus. In hac habitasse platea dictumst. In posuere ultricies nulla, eu interdum erat rhoncus ac. Vivamus rutrum porta interdum. Nulla pulvinar dui quis velit varius tristique dignissim sem luctus. Aliquam ac velit enim. Sed sed nisi faucibus ipsum congue lacinia. Morbi id mi in lectus vehicula dictum vel sed metus. Sed commodo lorem non nisl vulputate elementum. Fusce nibh dui, auctor a rhoncus eu, rhoncus eu eros."]

   [:paragraph
    "Nulla pretium ornare nisi at pulvinar. Praesent lorem diam, pulvinar nec scelerisque et, mattis vitae felis. Integer eu justo sem, non molestie nisl. Aenean interdum erat non nulla commodo pretium. Quisque egestas ullamcorper lacus id interdum. Ut scelerisque, odio ac mollis suscipit, libero turpis tempus nulla, placerat pretium tellus purus eu nunc. Donec nec nisi non sem vehicula posuere et eget sem. Aliquam pretium est eget lorem lacinia in commodo nisl laoreet. Curabitur porttitor dignissim eros, nec semper neque tempor non. Duis elit neque, sagittis vestibulum consequat ut, rhoncus sed dui."]

   [:anchor {:style {:size 15} :leading 20} "some anchor"]

   [:anchor [:phrase {:style "bold"} "some anchor phrase"]]

   [:anchor "plain anchor"]

   [:chunk {:style "bold"} "small chunk of text"]

   [:phrase "some text here"]

   [:phrase {:style "italic" :size 18 :family "helvetica" :color [0 255 221]} "Hello Clojure!"]

   [:chapter [:paragraph "Second Chapter"]]

   [:paragraph {:keep-together true :indent 20} "a fine paragraph"]

   [:list {:roman true} [:chunk {:style "bold"} "a bold item"] "another item" "yet another item"]

   [:chapter "Charts"]

   [:chart
    {:type :bar-chart :title "Bar Chart" :x-label "Items" :y-label "Quality"}
    [2 "Foo"]
    [4 "Bar"]
    [10 "Baz"]]

   [:chart
    {:type :line-chart :title "Line Chart" :x-label "checkpoints" :y-label "units"}
    ["Foo" [1 10] [2 13] [3 120] [4 455] [5 300] [6 600]]
    ["Bar" [1 13] [2 33] [3 320] [4 155] [5 200] [6 300]]]

   [:chart {:type :pie-chart :title "Big Pie"} ["One" 21] ["Two" 23] ["Three" 345]]

   [:chart
    {:type :line-chart
     :time-series true
     :title "Time Chart"
     :x-label "time"
     :y-label "progress"}
    ["Incidents"
     ["2011-01-03-11:20:11" 200]
     ["2011-01-03-11:25:11" 400]
     ["2011-01-03-11:35:11" 350]
     ["2011-01-03-12:20:11" 600]]]

   [:chapter "Graphics2D"]

   [:paragraph
    "Tree Attribution: "
    [:anchor
     {:style {:color [0 0 200]}
      :target "http://www.curiousattemptbunny.com/2009/01/simple-clojure-graphics-api.html"}
     "http://www.curiousattemptbunny.com/2009/01/simple-clojure-graphics-api.html"]]

   [:graphics {:translate [150 300] :rotate (radians -90)}
     (fn [g2d]
       (.setColor g2d java.awt.Color/GREEN)
       (draw-tree g2d 50 10))]

   [:chart {:type :pie-chart
            :title "Vector Pie"
            :vector true
            :width 300 :height 250
            :translate [270 100] }
    ["One" 21] ["Two" 23] ["Three" 345]]

   [:chart
    {:type :line-chart
     :title "Vector Line Chart"
     :x-label "checkpoints"
     :y-label "units"
     :vector true
     :width 500 :height 300
     :translate [50 400]}
    ["Foo" [1 10] [2 13] [3 120] [4 455] [5 300] [6 600]]
    ["Bar" [1 13] [2 33] [3 320] [4 155] [5 200] [6 300]]]

   [:chapter "Embedded SVG"]

   [:paragraph
    "Attribution: "
    [:anchor
     {:style {:color [0 0 200]}
      :target "https://en.wikipedia.org/wiki/File:Example.svg"}
     "https://en.wikipedia.org/wiki/File:Example.svg"]]

   [:svg {:under true :translate [0 270] :scale 0.95}
      (clojure.java.io/file "test/Example.svg")]

   [:pagebreak]

   [:paragraph
    "Attribution: "
    [:anchor
     {:style {:color [0 0 200]}
      :target "https://commons.wikimedia.org/wiki/SVG_examples"}
     "https://commons.wikimedia.org/wiki/SVG_examples"]]

   [:svg {}
     "<?xml version=\"1.0\"?>
      <!DOCTYPE svg PUBLIC \"-//W3C//DTD SVG 1.1//EN\" \"http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd\">
      <svg xmlns=\"http://www.w3.org/2000/svg\" width=\"467\" height=\"462\">
        <rect x=\"80\" y=\"60\" width=\"250\" height=\"250\" rx=\"20\" style=\"fill:#ff0000; stroke:#000000;stroke-width:2px;\" />
        <rect x=\"140\" y=\"120\" width=\"250\" height=\"250\" rx=\"40\" style=\"fill:#0000ff; stroke:#000000; stroke-width:2px; fill-opacity:0.7;\" />
      </svg>"]

   [:svg {:translate [100 450]}
     "<?xml version=\"1.0\" encoding=\"UTF-8\"?>
      <!DOCTYPE svg>
      <svg xmlns=\"http://www.w3.org/2000/svg\" width=\"304\" height=\"290\">
         <path d=\"M2,111 h300 l-242.7,176.3 92.7,-285.3 92.7,285.3 z\"
               style=\"fill:#FB2;stroke:#F00;stroke-width:3;stroke-linejoin:round\"/>
      </svg>"]]

  "example.pdf")
```

# Users

* [UHN](http://www.simspartners.ca/ourPartners/uhn.aspx) uses clj-pdf to generate reports for advanced clinical documentation.
* [SoftAddicts](http://www.softaddicts.ca/) process HL7 results with a mix of discrete values and images to produce a PDF in real time using clj-pdf.

Let me know if you find this library useful or if you have any suggestions.

# License
***

Copyright © 2015 Dmitri Sotnikov

Distributed under LGPL 3
