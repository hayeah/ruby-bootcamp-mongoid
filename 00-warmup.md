TODO

should include a section to get mongo installed

# Objectives

In this warmup exercise, you'll

+ create a ruby gem with bundler
+ build and install the ruby gem locally
+ push project to Github

If you get stuck, don't be afraid to ask on the [team microblog](http://rubybootcamp01.wordpress.com/).

# Create a new Ruby gem

```
> bundle gem my_mongoid
      create  my_mongoid/Gemfile
      create  my_mongoid/Rakefile
      create  my_mongoid/LICENSE.txt
      create  my_mongoid/README.md
      create  my_mongoid/.gitignore
      create  my_mongoid/my_mongoid.gemspec
      create  my_mongoid/lib/my_mongoid.rb
      create  my_mongoid/lib/my_mongoid/version.rb
```

# Things to notice

Gem dependencies are in `my_mongoid.gemspec`. Gemfile loads from gemspec.

The project version is in `lib/my_mongoid/version.rb` as a Ruby constant string.

# Commit the initial project

If you do run `git log` now, you get this error:

```
> git log
fatal: bad default revision 'HEAD'
```

This is because the project files are at the staging area, and the git repository has no commits yet. Commit now.

```
> git commit -m "init project"
[master (root-commit) 56610d3] init project
 8 files changed, 104 insertions(+)
 create mode 100644 .gitignore
 create mode 100644 Gemfile
 create mode 100644 LICENSE.txt
 create mode 100644 README.md
 create mode 100644 Rakefile
 create mode 100644 lib/mymongoid.rb
 create mode 100644 lib/mymongoid/version.rb
 create mode 100644 mymongoid.gemspec
```

# Gem release rake tasks

The bundler gem includes a few rake tasks to help you manage gem releases.

To list the commands, do:

```
> rake -T
rake build    # Build mymongoid-0.0.1.gem into the pkg directory
rake install  # Build and install mymongoid-0.0.1.gem into system gems
rake release  # Create tag v0.0.1 and build and push mymongoid-0.0.1.gem to Rubygems
```

Now try edit `mymongoid/lib/mymongoid/version.rb`. Change `0.0.1` to `0.0.2`. Rerun `rake -T`, and see what changed.

# The gemspec

Gemspec reference, [here](http://guides.rubygems.org/specification-reference/).

# Build the gem

Now, try to build the gem using the rake task `rake build`

```
> rake build
rake aborted!
ERROR:  While executing gem ... (Gem::InvalidSpecificationException)
    "FIXME" or "TODO" is not a description
```

Please fix it by changing the gemspec. You should get the output

```
> rake build
my_mongoid 0.0.1 built to pkg/my_mongoid-0.0.1.gem.
```

Commit your changes.

# What's in a ruby gem

Now that you have a gem at `pkg/my_mongoid-0.0.1.gem` , what the heck is it?

```
> file pkg/my_mongoid-0.0.1.gem
pkg/my_mongoid-0.0.1.gem: POSIX tar archive
```

It's a tar archive. Let's see its content

```
> tar -tf pkg/my_mongoid-0.0.1.gem
metadata.gz
data.tar.gz
checksums.yaml.gz
```

The gem has 3 files. The metadata, the packaged content (data.tar.gz), and a checksum file.

The metadata is basically the evaluated gemspec file. To see it:

```
> tar -xOf pkg/my_mongoid-0.0.1.gem metadata.gz | gzip -d
--- !ruby/object:Gem::Specification
name: my_mongoid
version: !ruby/object:Gem::Version
  version: 0.0.1
platform: ruby
authors:
- Howard Yeh
autorequire:
bindir: bin
cert_chain: []
...
```

Let's take that command apart.

1. `tar -xOf pkg/my_mongoid-0.0.1.gem metadata.gz` extracts metadata.gz
2. the -O flag specifies that the extracted be sent to stdout
3. the stdout of the tar command is piped to gzip command
4. `gzip -d` decompresses metadata.gz, and output the stdout

The result is the decompressed gem metadata.

To see the the files packaged by the gem, do

```
> tar -xOf pkg/my_mongoid-0.0.1.gem data.tar.gz | tar -ztf -
.gitignore
Gemfile
LICENSE.txt
README.md
Rakefile
lib/my_mongoid.rb
lib/my_mongoid/version.rb
my_mongoid.gemspec
```

Can you understand this command? If you don't understand it, go ask on [team microblog](http://rubybootcamp01.wordpress.com/).

# How to install the gem locally

Install

```
gem install pkg/my_mongoid-0.0.1.gem --local
Successfully installed my_mongoid-0.0.1
Parsing documentation for my_mongoid-0.0.1
Done installing documentation for my_mongoid after 0 seconds
1 gem installed
```

Where is it installed?

```
> gem which my_mongoid
/Users/howard/.rvm/gems/ruby-2.0.0-p247/gems/my_mongoid-0.0.1/lib/my_mongoid.rb
```

Does it work?

```
> ruby -r my_mongoid -e 'puts MyMongoid::VERSION'
0.0.1
```

Great job! You've created a Ruby gem.

# Push to github

Finally, let's push the project to Github. Let's give the repo the name "my_mongoid".

Go to [https://github.com/new](https://github.com/new) to create a new Github project.

You've finished the bootcamp warmup : )