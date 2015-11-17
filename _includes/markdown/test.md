
#### TLDR -- `ng-lazy-show` gives the benefits of both `ng-if` and `ng-show` -- <a href="https://gist.github.com/accolver/4987806b026fd8a03568" target="_blank">gist</a>

## The Problem

As Angular apps grow, the sheer complexity of the two-way binding begins to slow down your application. As Misko Hevery (one of the AngularJS developers) has pointed out, after ~2000 watches an app begins to slow to an unacceptable rate.

Todd Motto talks about some great optimizations to speed up your Angular app. As our app at <a href="http://www.lendio.com" target="_blank">Lendio</a> began to grow, we needed to optimize. We applied some of the techniques Todd discussed, but our performance could still improve more.

Using Chrome's JavaScript profiler, I identified a large directive that was taking ~500 ms to load. This directive is used for optional, advanced filtering in a list view. It was using `ng-show` to allow the user to toggle showing the filters.


```
<button ng-click="showFilters = !showFilters">Toggle Filters</button>
```

#### Attempt 1


```
<pre><code>
<div ng-show="showFilters" lendio-business-filters></div>
</code></pre>
```

Using `ng-show` causes Angular to eager load the directive -- compiling it and all its dependencies. This made the button to toggle the available filters very responsive but added a half-second overhead every time the view was loaded -- even if the user never clicked to see the filters! So I tried ng-if...

#### Attempt 2
```
<div ng-if="showFilters" lendio-business-filters></div>
```
This eliminated the overhead when the view loaded, but incurred the half-second delay EVERY time the visibility of the filters was toggled. This happens because when the ng-if expression is falsey, the DOM element is completely removed. And when the expression evaluates truthy again, Angular has to again compile the directive.

## Solution
A mix of `ng-show` and `ng-if` is what we need. We need to delay the compilation of the directive because, heck, it may never be used, and once it's been loaded, we don't want to have to compile the directive all over again.

```
<div ng-lazy-show="showFilters" lendio-business-filters></div>
```

`ng-lazy-show` acts like an `ng-if` until its expression evaluates truthy the first time. After that, it behaves as a regular `ng-show`. `ng-lazy-show` is currently not a part of Angular. You can reference the code below or the gist if you would like to implement this in your own app.


```
var ngLazyShowDirective = ['$animate', function ($animate) {

  return {
    multiElement: true,
    transclude: 'element',
    priority: 600,
    terminal: true,
    restrict: 'A',
    link: function ($scope, $element, $attr, $ctrl, $transclude) {
      var loaded;
      $scope.$watch($attr.ngLazyShow, function ngLazyShowWatchAction(value) {
        if (loaded) {
          $animate[value ? 'removeClass' : 'addClass']($element, 'ng-hide');
        }
        else if (value) {
          loaded = true;
          $transclude(function (clone) {
            clone[clone.length++] = document.createComment(' end ngLazyShow: ' + $attr.ngLazyShow + ' ');
            $animate.enter(clone, $element.parent(), $element);
            $element = clone;
          });
        }
      });
    }
  };

}];

angular.module('yourModule').directive('ngLazyShow', ngLazyShowDirective);
```

Comments and suggestions are welcome here or on the <a href="https://gist.github.com/accolver/4987806b026fd8a03568" target="_blank">gist</a>. Thanks for reading!
