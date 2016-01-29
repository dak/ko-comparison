# Knockout Performance and Code Comparison

Below are screen grabs of the performance profile of toggling between subjects on the subjects page with and without Knockout.JS.
Tests were run with a fresh page reload on my 2012 15" MacBookPro in Chrome 48.

## Knockout Performance

### With Knockout

<p align="center">
    <img height="203" width="584" src="https://raw.githubusercontent.com/dak/ko-comparison/master/ko.png">
</p>

### Without Knockout

<p align="center">
    <img height="204" width="584" src="https://raw.githubusercontent.com/dak/ko-comparison/master/no-ko.png">
</p>

### Analysis

 - KO gets about **12 times** less frames per second, and is being flagged by Chrome as indicating [jank](https://developers.google.com/web/fundamentals/performance/rendering/).
 - KO takes **20 times** longer to finish rendering.
 - KO script execution takes more than **30 times** longer to complete the equivalent DOM update.

This difference is not just theoretical.  On slower devices (and virtually every mobile device),
it will cause noticeable screen flicker and will feel less responsive.

## Code Comparison

### 1. Row of Buttons

#### With Knockout

```html
<div class="button-row">
    <div class="filter-buttons" data-bind="foreach: filterButtons">
        <div class="filter-button" data-bind="text: $data,
         click: $parent.selectedFilter,
         css: {selected: $parent.isSelectedFilter($data)}">
        </div>
    </div>
</div>
```

#### Without Knockout

```handlebars
<div class="book-menu">
    <ul>
        <li>View All</li>
        {{#each subjects}}
        <li>{{this}}</li>
        {{/each}}
    </ul>
</div>
```

#### Analysis

The Knockout version of the template is not only more code and more difficult
to read, as a result of all the `data-bind` properties attached to DOM elements,
but it also injects logic and is calling functions within the HTML itself, for
example: `css: {selected: $parent.isSelectedFilter($data)}`.  It also attaches
a click handler with `click: $parent.selectedFilter`.

These snippets of code not only make the code more difficult to read, but are also
anti-patterns.  That degree of logic should not be mixed into the template itself,
since it makes the code harder to read, debug, and maintain.

By comparison, the version without Knockout has only a simple loop, and that loop
is also much easier to identify by eye as it is clearly separated from the actual
DOM elements.

### 2. List of Books

#### With Knockout

```html
<div class="book-viewer" data-bind="foreach: filteredBooks()">
    <div class="subject" data-bind="text: category"></div>
    <div class="row-of-covers" data-bind="foreach: books">
        <div class="book-info" data-bind="visible: isSelected()">
            <div class="get-this-title" data-bind="render: $root.subview">
            </div>
        </div>
        <div class="cover" data-bind="click: $root.selectedBook.toggle,
         css: {selected: isSelected()}">
            <img data-bind="attr: {src: book.coverUrl}" />
            <div class="arrow-up"></div>
        </div>
    </div>
</div>
```

#### Without Knockout

```handlebars
{{#each books}}
<div class="subject">{{@key}}</div>
<ul>
    {{#each this}}
    <li>
        <div class="book">
            <img src="{{cover}}">
        </div>
    </li>
    {{/each}}
</ul>
{{/each}}
```

#### Analysis

Once again, the Knockout version has significant logic embedded into it,
explicitly calling functions in three separate locations, as well as other
implicit calls, such as `data-bind="render: $root.subview"` and attaching
another click handler (`data-bind="click: $root.selectedBook.toggle"`).

It's also injecting the `src` property into an `img` tag through a separate
binding, which if nothing else affects code clarity:
`<img data-bind="attr: {src: book.coverUrl}" />`.

Finally, all of these bindings also cause Knockout to refresh the DOM multiple
times for each binding that causes a change, which is why the rendering
performance is so much worse (and the scripting performance is worse because
of all the extra code that is executed to process all these bindings).

### 3. Subjects Page

#### With Knockout

```js
import BaseView from '~/helpers/backbone/view';
import GetThisTitle from '~/components/get-this-title/get-this-title';
import {props} from '~/helpers/backbone/decorators';
import {template} from './subjects.hbs';
import ko from 'knockout';

let Region = new BaseView().regions.self.constructor;

ko.bindingHandlers.render = {
    init: function (el, val) {
        let region = new Region(el),
            constructorFn = val();

        region.show(new constructorFn());
    }
};

function book(title, categories) {
    return {
        title,
        categories,
        coverUrl: `https://placeholdit.imgix.net/~text?txtsize=33&txt=${title}&w=200&h=180`
    };
}

function viewModel(subview) {
    let filterButtons = [
            'View All', 'Math', 'Science', 'Social Sciences', 'History', 'AP'
        ],
        selectedFilter = ko.observable('View All'),
        isSelectedFilter = (filterButton) => filterButton === selectedFilter(),
        books = [
            book('Math 1', ['Math']),
            book('Math AP', ['Math', 'AP']),
            book('Math 2', ['Math']),
            book('Math 3', ['Math']),
            book('History 1', ['History']),
            book('History 2', ['History']),
            book('Science 1', ['Science']),
            book('Science 2', ['Science']),
            book('Science AP', ['Science', 'AP']),
            book('Social', ['Social Sciences'])
        ],
        filteredCategories = () => {
            if (selectedFilter() === 'View All') {
                return filterButtons.filter((c) => c !== 'View All');
            }
            return [selectedFilter()];
        },
        selectedBook = ko.observable(),
        filteredBooks = () =>
            filteredCategories().map(function (c) {
                let result = {
                    category: c,
                    books: books.filter(
                        (b) => b.categories.indexOf(c) >= 0).map(function (b) {
                            let bResult = {
                                book: b,
                                isSelected: () => selectedBook() === bResult
                            };

                            return bResult;
                        }),
                    isSelectedCategory: () => result.books.indexOf(selectedBook()) >= 0
                };

                return result;
            });

        selectedBook.toggle = (item) => {
            if (selectedBook() !== item) {
                selectedBook(item);
            } else {
                selectedBook(null);
            }

        };

    return {
        filterButtons,
        selectedFilter,
        isSelectedFilter,
        selectedBook,
        filteredBooks,
        subview
    };
}

@props({
    template: template
})
export default class Subjects extends BaseView {
    onRender() {
        this.el.classList.add('text-content');
        ko.applyBindings(viewModel(GetThisTitle, this.el));
    }
}
```

#### Without Knockout

```js
import BaseView from '~/helpers/backbone/view';
import {on, props} from '~/helpers/backbone/decorators';
import {template} from './subjects.hbs';
import library from '~/models/library';
import BooksList from './books/books';

@props({
    template: template,
    templateHelpers: {
        subjects: library.subjects
    },
    regions: {
        books: '.books'
    }
})
export default class Subjects extends BaseView {

    onRender() {
        this.booksList = new BooksList();
        this.regions.books.show(this.booksList);
    }

    @on('click .book-menu li')
    selectSubject(e) {
        this.booksList.setCategory(e.target.textContent);
    }

}
```

#### Analysis

One could argue that this comparison is deceptive, since the version
without Knockout has factored out the Model and Book List into separate modules;
however, I believe that actually raises the first issue with the Knockout
example.

Just as the Knockout DOM examples above mixed logic into the DOM, the Knockout
ViewModel example mixes code that should be separate modules into one single
large file.  For example, the Model is entirely contained within the Knockout
ViewModel code.  This also encourages shortcuts, such as further hardcoding
the list of subjects, whereas the version without Knockout calculates it
dynamically (see the full comparisons below for the Model and BookList code).

Separating the code logically into separate modules not only makes the code easier
to read, debug, and maintain, but it also makes the code more reusable.  Any other
module can easily use the model representing the Library of books in the
version without Knockout, whereas that is effectively impossible in the
Knockout example.

Additionally, all of these extra advantages come at very little cost, as the
number of lines of code is nearly identical, even though the version without
Knockout has some additional boilerplate (like importing dependencies).

### Conclusion

Some of the above shortcomings could be overcome by refactoring the Knockout
example, but other shortcomings, including the performance and template
clarity, will never be comparable.

Due to the numerous cons of including Knockout (not to mention it's an
additional 55KB library), and the fact that it has at best a negligible impact
on the amount of code that has to be written (and may not actually reduce
LoC at all, while simultaneously reducing code clarity and maintainability),
I recommend against including Knockout.

### Note:
See Full comparisons at:

Without Knockout: https://github.com/openstax/os-webview/compare/subjects-no-knockout

With Knockout: https://github.com/openstax/os-webview/compare/subjects
