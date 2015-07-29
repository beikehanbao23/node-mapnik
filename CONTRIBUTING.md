# Contributing

General guidelines for contributing to node-mapnik

## Coding Conventions

Mapnik is written in C++, and we try to follow general coding guidelines.

If you see bits of code around that do not follow these please don't hesitate to flag the issue or correct it yourself.

#### Prefix cmath functions with std::

The avoids ambiguity and potential bugs of using old C library math directly.

So always do `std::abs()` instead of `abs()`. Here is a script to fix your code in one fell swoop:


```sh
DIR=./bindings
for i in {abs,fabs,tan,sin,cos,floor,ceil,atan2,acos,asin}; do
    find $DIR -type f -name '*.cpp' -or -name '*.h' -or -name '*.hpp' | xargs perl -i -p -e "s/ $i\(/ std::$i\(/g;"
    find $DIR -type f -name '*.cpp' -or -name '*.h' -or -name '*.hpp' | xargs perl -i -p -e "s/\($i\(/\(std::$i\(/g;"
done
```

#### Avoid boost::lexical_cast

It's slow both to compile and at runtime.

#### Avoid sstream objects if possible

They should never be used in performance critical code because they trigger std::locale usage
which triggers locks

#### Spaces not tabs, and avoid trailing whitespace

#### Indentation is four spaces

#### Use C++ style casts

    static_cast<int>(value); // yes

    (int)value; // no


#### Use const keyword after the type

    std::string const& variable_name // preferred, for consistency

    const std::string & variable_name // no


#### Pass built-in types by value, all others by const&

    void my_function(int double val); // if int, char, double, etc pass by value

    void my_function(std::string const& val); // if std::string or user type, pass by const&

#### Use unique_ptr instead of new/delete

#### Use std::copy instead of memcpy

#### When to use shared_ptr and unique_ptr

Sparingly, always prefer passing objects as const& except where using share_ptr or unique_ptr express more clearly your intent. See http://herbsutter.com/2013/06/05/gotw-91-solution-smart-pointer-parameters/ for more details.

#### Shared pointers should be created with std::make_shared.

#### Use assignment operator for zero initialized numbers

    double num = 0; // please

    double num(0); // no


#### Function definitions should not be separated from their arguments:

    void foo(int a) // please

    void foo (int a) // no


#### Separate arguments by a single space:

    void foo(int a, float b) // please

    void foo(int a,float b) // no


#### Space between operators:

    if (a == b) // please

    if(a==b) // no


#### Braces should always be used:

    if (!file)
    {
        throw mapnik::datasource_exception("not found"); // please    
    }

    if (!file)
        throw mapnik::datasource_exception("not found"); // no


#### Braces should be on a separate line:

    if (a == b)
    {
        int z = 5;
        // more...
    }


#### Prefer `empty()` over `size() == 0` if container supports it

This avoids implicit conversions to bool and reduces compiler warnings.

    if (container.empty()) // please

    if (container.size() == 0) // no


### Other C++ style resources

Many also follow the useful [Google style guide](http://google-styleguide.googlecode.com/svn/trunk/cppguide.xml) which mostly fits our style. However, Google obviously has to maintain a lot of aging codebases. Mapnik can move faster, so we don't follow all of those style recommendations.

## Testing

In order for any code to be pulled into master it must contain tests for **100%** of all lines. The only lines that are not required to be tested are those that cover extreme cases which can not be tested with regularity, such as race conditions. 

If this case does occur you can put a comment block such as shown below to exclude the lines from test coverage.

```C++
// LCOV_EXCL_START
can_not_reach_code();
// LCOV_EXCL_END
```

## Releasing

To release a new node-mapnik version:

### Create a branch and publish binaries

**1)** Create a branch for your work

**2)** Updating the Mapnik SDK used for binaries

If your node-mapnik release requires a new Mapnik version, then a new Mapnik SDK would need to be published first.

To get a new Mapnik SDK published for Unix:

  - The Mapnik master branch must be production ready.
  - Or you can build an SDK from a testing branch. In this case the `MAPNIK_BRANCH` value [here](https://github.com/mapnik/mapnik-packaging/blob/master/.travis.yml#L8) can be edited.
  - Next you would commit to the `mapnik-packaging` repo with the message of `-m "[publish]"`.
  - Watch the end of the travis logs to get the url of the new SDK (https://travis-ci.org/mapnik/mapnik-packaging/builds/47075304#L4362)

Then, back at node-mapnik, take the new SDK version (which is generated by `git describe` and add it to the `MAPNIK_GIT` variable in the `.travis.yml`.

To get a new Mapnik SDK published for Windows:

  - The scripts at https://github.com/BergWerkGIS/mapnik-dependencies are used
  - The do not run fast enough to work on appveyor so we plan aws automation to get this working.
  - In the meantime ping @BergWerkGIS to run them and provide a new SDK.

Then, back at node-mapnik, take the new SDK version (which is generated by `git describe` and add it to the `MAPNIK_GIT` variable in the `appveyor.yml`.

**3)** Make sure all tests are passing on travis and appveyor for your branch. Check the links at https://github.com/mapnik/node-mapnik/blob/master/README.md#node-mapnik.

**4)** Within your branch edit the `version` value in `package.json` to something unique that will not clash with existing released versions. For example if the current release (check this via https://github.com/mapnik/node-mapnik/releases) is `3.1.4` then you could rename your `version` to `3.1.5-alpha` or `3.1.5-branchname`.

**5)** Commit the new version and publish binaries

Do this like:

```sh
git commit package.json -m "bump to v3.1.5-alpha [publish binary]"
```

What if you already committed the `package.json` bump and you have no changes to commit but want to publish binaries. In this case you can do:

```sh
git commit --allow-empty -m "[publish binary]"
```

If you need to republish binaries you can do this with the command below, however this should not be a common thing for you to do!

```sh
git commit --allow-empty -m "[republish binary]"
```

Note: NEVER republish binaries for an existing released version.

**6)** Test your binaries

Lots of ways to do this of course. Just updating a single dependency of node-mapnik is a good place to start: https://www.npmjs.com/browse/depended/mapnik

But the ideal way is to test a lot at once: enter mapnik-swoop.

### mapnik-swoop

Head over to https://github.com/mapbox/mapnik-swoop and create a branch that points [the package.json](https://github.com/mapbox/mapnik-swoop/blob/master/package.json#L14) at your working node-mapnik version.

Ensure that all tests are passing. Only ignore failing tests for dependencies if you can confirm with the downstream maintainers of the modules that those tests are okay to fail and unrelated to your node-mapnik changes. You can check [recent builds](https://travis-ci.org/mapbox/mapnik-swoop/builds) to see if all builds were green and passing before your change. If they were red and failing before then try to resolve those issues before testing your new node-mapnik version.

**7)** Official release

An official release requires:

 - Updating the CHANGELOG.md
 - Publishing new binaries for a non-alpha version like `3.1.5`. So you'd want to merge your branch and then edit the `version` value in package json back to a decent value for release.
 - Create a github tag like `git tag 3.1.5 -m "v3.1.5"`
 - Test mapnik-swoop again for your new tagged version
 - Ensure you have a clean checkout (no extra files in your check that are not known by git). You need to be careful, for instance, to avoid a large accidental file being packaged by npm. You can get a view of what npm will publish by running `make testpack`
 - Then publish the module to npm repositories by running `npm publish`

### Documentation

node-mapnik is documented with [JSDoc](http://usejsdoc.org/) comments embedded
in the C++ code and formatted into HTML with [documentationjs](http://documentation.js.org/).

To update the [hosted documentation](http://mapnik.org/node-mapnik/documentation/):

* Switch to the `gh-pages` branch: `git checkout gh-pages`
* Regenerate documentation: `npm run docs`
* Add the changed files in the `documentation` path
* Commit the changes