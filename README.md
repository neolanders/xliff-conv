# xliff-conv (work in progress)

XLIFF to/from JSON converter for Polymer [i18n-behavior](https://github.com/t2ym/i18n-behavior)

## Features

- Update bundle.*.json values with those from XLIFF
- Generate XLIFF from bundles
- Map todo operations in bundles onto XLIFF states
- Update todo operations in bundles with XLIFF states
- [UMD](https://github.com/ruyadorno/generator-umd) support

## TODOs

- Test suites on node.js and browsers

## Install

### For Node.js

```javascript
    npm install --save-dev xliff-conv
```

[Quick Tour](https://github.com/t2ym/polymer-starter-kit-i18n#quick-tour) with polymer-starter-kit-i18n

### For Browsers

```javascript
	bower install --save xliff-conv
```

## Import

### On Node.js

```javascript
	var XliffConv = require('xliff-conv');
```

### On Browsers

```html
	<script src="path/to/bower_components/xliff-conv/xliff-conv.js"></script>
```

## Examples

### Import XLIFF task on gulp

#### Note: This task has to be processed before [Leverage task with unbundle](https://github.com/t2ym/gulp-i18n-leverage#leverage-task-with-unbundle) to pick up outputs of this task.

#### Input:
  - Next XLIFF files in source
  - Current bundle JSON files in source (as output templates)

#### Output:
  - Overwritten bundle JSON files in source

```javascript
    var gulp = require('gulp');
    var JSONstringify = require('json-stringify-safe');
    var through = require('through2');
    var XliffConv = require('xliff-conv');

    // Import bundles.{lang}.xlf
    gulp.task('import-xliff', function () {
      var xliffPath = path.join('app', 'xliff');
      var xliffConv = new XliffConv();
      return gulp.src([
          'app/**/xliff/bundle.*.xlf'
        ])
        .pipe(through.obj(function (file, enc, callback) {
          var bundle, bundlePath;
          var base = path.basename(file.path, '.xlf').match(/^(.*)[.]([^.]*)$/);
          var xliff = String(file.contents);
          if (base) {
            try {
              bundlePath = path.join(file.base, 'locales', 'bundle.' + base[2] + '.json');
              bundle = JSON.parse(stripBom(fs.readFileSync(bundlePath, 'utf8')));
              xliffConv.parseXliff(xliff, { bundle: bundle }, function (output) {
                file.contents = new Buffer(JSONstringify(output, null, 2));
                file.path = bundlePath;
                callback(null, file);
              });
            }
            catch (ex) {
              callback(null, file);
            }
          }
          else {
            callback(null, file);
          }
        }))
        .pipe(gulp.dest('app'))
        .pipe($.size({
          title: 'import-xliff'
        }));
    });
```

### Export XLIFF task on gulp

#### Input:
  - Next bundles object in gulpfile.js

#### Output:
  - bundle.{lang}.xlf XLIFF in DEST_DIR/xliff

```javascript
    var gulp = require('gulp');
    var through = require('through2');
    var XliffConv = require('xliff-conv');

    var bundles; // bundles object generated by preprocess and leverage tasks

    // Generate bundles.{lang}.xlf
    gulp.task('export-xliff', function (callback) {
      var DEST_DIR = 'dist';
      var srcLanguage = 'en';
      var xliffPath = path.join(DEST_DIR, 'xliff');
      var xliffConv = new XliffConv();
      var promises = [];
      try {
        fs.mkdirSync(xliffPath);
      }
      catch (e) {
      }
      for (var lang in bundles) {
        if (lang) {
          (function (destLanguage) {
            promises.push(new Promise(function (resolve, reject) {
              xliffConv.parseJSON(bundles, {
                srcLanguage: srcLanguage,
                destLanguage: destLanguage
              }, function (output) {
                fs.writeFile(path.join(xliffPath, 'bundle.' + destLanguage + '.xlf'), output, resolve);
              });
            }));
          })(lang);
        }
      }
      Promise.all(promises).then(function (outputs) {
        callback();
      });
    });
```

## API

### Constructor

`var xliffConv = new XliffConv(options)`

#### `options` object

- date: Date, default: new Date() - date attribute value for XLIFF
- xliffStates: Object, default: xliffConv.xliffStates.default - todo.op to XLIFF state mapping table

#### `xliffConv.xliffStates` object - default value for options.xliffStates

```javascript
  {
    'default': {
      'add'    : [ 'new' ],
      'replace': [ 'needs-translation', 'needs-adaptation', 'needs-l10n', '' ],
      'review' : [ 'needs-review-translation', 'needs-review-adaptation', 'needs-review-l10n' ],
      'default': [ 'translated', 'signed-off', 'final', '' ]
    }
  }
```

### `xliffConv.parseXliff(xliff, options, callback)` method

- xliff: String, XLIFF as a string
- options: Object, options.bundle as target bundle JSON object
- callback: Function, callback(output) with output JSON object

### `xliffConv.parseJSON(bundles, options, callback)` method

- bundles: Object, bundles object
- options.srcLanguage: String, default: 'en' - source language
- options.destLanguage: String, default: 'fr' - target language
- callback: Function, callback(output) with output XLIFF as a string

### Custom XLIFF restype attributes

| restype          | JSON type | Note                       |
|:-----------------|:----------|:---------------------------|
| x-json-string    | string    | Omitted                    |
| x-json-boolean   | boolean   | "true" or "false" in value |
| x-json-number    | number    |                            |
| x-json-object    | object    | Unused for now             |
| x-json-undefined | undefined | Empty string in value      |

### Default Mapping of todo.op and XLIFF states

#### JSON -> XLIFF

| todo.op | XLIFF state              |
|:--------|:-------------------------|
| add     | new                      |
| replace | needs-translation        |
| review  | needs-review-translation |
| N/A     | translated               |

#### XLIFF -> JSON

| XLIFF state              | approved  | todo.op | Note        |
|:-------------------------|:----------|:--------|:------------|
| new                      | no or N/A | add     |             |
| needs-translation        | no or N/A | replace |             |
| needs-adaptation         | no or N/A | replace |             |
| needs-l10n               | no or N/A | replace |             |
| needs-review-translation | no or N/A | review  |             |
| needs-review-adaptation  | no or N/A | review  |             |
| needs-review-l10n        | no or N/A | review  |             |
| translated               | yes       | N/A     | Remove todo |
| signed-off               | yes       | N/A     | Remove todo |
| final                    | yes       | N/A     | Remove todo |
| N/A                      | *         | N/A     | Remove todo |

## License

[BSD-2-Clause](https://github.com/t2ym/xliff-conv/blob/master/LICENSE.md)
