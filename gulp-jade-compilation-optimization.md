Optimizations for compiling Jade in a gulp task
===============================================

_Using Jade inheritance and construct cache for all the dependencies._

For this task we will use these different packages and global variables
```js
var 
  fs = require('fs'),
  path = require('path'),
  JadeInheritance = require('jade-inheritance'),
  gulp = require('gulp'),
  jade = require('gulp-jade'),
  plumber = require('gulp-plumber'),
  watch = require("gulp-watch"),
  chalk = require("chalk"),
  args = require('yargs').argv,

  sources = 'source/', // the folder where all your templates are
  dependenciesSave = 'jadeDependencies.json', // the file where we will save the dependencies of each template of the app
  dependantFiles = '',
  isProduction = args.type === 'production';
```

And a function to debounce the `onChange` event
```js
var debounce = function(func, threshold, execAsap) {
  var timeout;
  timeout = null;
  return function() {
    var args, delayed, obj;
    args = 1 <= arguments.length ? __slice.call(arguments, 0) : [];
    obj = this;
    delayed = function() {
      if (!execAsap) {
        func.apply(obj, args);
      }
      return timeout = null;
    };
    if (timeout) {
      clearTimeout(timeout);
    } else if (execAsap) {
      func.apply(obj, args);
    }
    return timeout = setTimeout(delayed, threshold || 100);
  };
};
```


First you need to construct a task to watch your jade files
```js
// watch the jade files for any change
gulp.task('watchJade', function () {
  var jadeW,
      filtering,
      onChangeJade,
      changedFiles = [];

  jadeW = gulp.watch([sources + '**/*.jade'],{ maxListeners: 999 });
  jadeW.on('change', function(event) {
    if (event.type === 'changed') {
      changedFiles.push(event.path);
      onChangeJade();
    }
  });

  // ...

});
```

Create a function to filter only the *views*. Because you don't want to compile the other template that are only parts or layout used in other views.
```js
// ...

filtering = function (element, index) {
  var regex = new RegExp('^source/jade/(?:pages/.*)?[^/]*\.jade$', 'i');
  if (typeof element === 'string') {
    return regex.test(element);
  }
  return false
};
```

Then you need to go throw the inheritance array of your modified files and compile only the files needed.
If you are in development mode you will only compile the first *view* to check and debug. For production, you will compile all the *views* inherited.

```js    
// ...

onChangeJade = debounce(function () {
  console.log(chalk.green.bold('jade watch on change: start...'));
  console.time(chalk.green.bold('jade watch on change'));
  fs.exists(dependenciesSave, function (exists) {
    if (!exists) {
      // create the file if not exist
      fs.writeFile(dependenciesSave, '{}', function (err) {
          if (err) throw err;
       });
    }

    fs.readFile(dependenciesSave, function (err, data) {
      if (err) throw err;
      var dependantFilesTemp = '';
      var dependantFilesCache = JSON.parse(data) || {};
      // console.time('jade-inheritance total');

      changedFiles.forEach(function (filepath) {
        var basedir = process.cwd(),
            filename = path.relative(basedir, filepath),
            inheritance,
            jsonString;

        if (typeof dependantFilesCache[filename] === 'undefined') {
          inheritance = new JadeInheritance(filename, '.', {basedir: '.'});

          dependantFilesTemp = inheritance.files.filter(filtering);
          dependantFilesCache[filename] = dependantFilesTemp;
          jsonString = JSON.stringify(dependantFilesCache);
          fs.writeFile(dependenciesSave, jsonString, function (err) {
            if (err) throw err;
            console.log(chalk.blue('The Jade dependencies for the file \''+chalk.bold(filename)+'\' is saved in '+chalk.bold(dependenciesSave)+' !'));
          });

        }
        else {
          console.log(chalk.blue('dependencies from cache '+ chalk.bold(dependenciesSave)));
        }

        dependantFilesTemp = JSON.parse(JSON.stringify(dependantFilesCache[filename]));

        if (!isProduction) {
          // reduce the compiled files to only one page for debugging
          dependantFiles = dependantFilesTemp.shift();
        }

        console.log(chalk.yellow('Let\' check this file to test and debug : ') + chalk.bgYellow.black(dependantFiles));
      });

      // console.timeEnd('jade-inheritance total');
    
      gulp.start('JadeDependant');

      changedFiles = [];
      console.timeEnd(chalk.green.bold('jade watch on change'));
    });
  });
}, 500);
```

Finally we need to create an another task to compile the jade files
```js
// compile only the dependant templates (used with the task 'watchJade')
gulp.task('JadeDependant', function () {
  if (dependantFiles != ''){ // dependantFiles is a global variable
    gulp.src(dependantFiles)
      .pipe(plumber())
      .pipe(jade({pretty: true, compileDebug:true}))
      .pipe(gulp.dest(sitePath));
  }
});
```