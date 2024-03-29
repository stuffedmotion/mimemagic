# CarrierWave

This gem provides a simple and extremely flexible way to upload files from Ruby applications.
It works well with Rack based web applications, such as Ruby on Rails.

[![Build Status](https://travis-ci.org/carrierwaveuploader/carrierwave.png?branch=master)](http://travis-ci.org/carrierwaveuploader/carrierwave)
[![Code Climate](https://codeclimate.com/github/carrierwaveuploader/carrierwave.png)](https://codeclimate.com/github/carrierwaveuploader/carrierwave)

## Information

* RDoc documentation [available on RubyDoc.info](http://rubydoc.info/gems/carrierwave/frames)
* Source code [available on GitHub](http://github.com/carrierwaveuploader/carrierwave)
* More information, known limitations, and how-tos [available on the wiki](https://github.com/carrierwaveuploader/carrierwave/wiki)

## Getting Help

* Please ask the [Google Group](http://groups.google.com/group/carrierwave) for help if you have any questions.
* Please report bugs on the [issue tracker](http://github.com/carrierwaveuploader/carrierwave/issues) but read the "getting help" section in the wiki first.

## Installation

Install the latest stable release:

	[sudo] gem install carrierwave

In Rails, add it to your Gemfile:

```ruby
gem 'carrierwave'
```

Finally, restart the server to apply the changes.

Note that CarrierWave is not compatible with Rails 2 as of version 0.5. If you want to use
Rails 2, please use the 0.4-stable branch on GitHub.

## Getting Started

Start off by generating an uploader:

	rails generate uploader Avatar

this should give you a file in:

	app/uploaders/avatar_uploader.rb

Check out this file for some hints on how you can customize your uploader. It
should look something like this:

```ruby
class AvatarUploader < CarrierWave::Uploader::Base
  storage :file
end
```

You can use your uploader class to store and retrieve files like this:

```ruby
uploader = AvatarUploader.new

uploader.store!(my_file)

uploader.retrieve_from_store!('my_file.png')
```

CarrierWave gives you a `store` for permanent storage, and a `cache` for
temporary storage. You can use different stores, including filesystem
and cloud storage.

Most of the time you are going to want to use CarrierWave together with an ORM.
It is quite simple to mount uploaders on columns in your model, so you can
simply assign files and get going:

### ActiveRecord

Make sure you are loading CarrierWave after loading your ORM, otherwise you'll
need to require the relevant extension manually, e.g.:

```ruby
require 'carrierwave/orm/activerecord'
```

Add a string column to the model you want to mount the uploader by creating
a migration:


	rails g migration add_avatar_to_users avatar:string
	rake db:migrate

Open your model file and mount the uploader:

```ruby
class User < ActiveRecord::Base
  mount_uploader :avatar, AvatarUploader
end
```

Now you can cache files by assigning them to the attribute, they will
automatically be stored when the record is saved.

```ruby
u = User.new
u.avatar = params[:file]
u.avatar = File.open('somewhere')
u.save!
u.avatar.url # => '/url/to/file.png'
u.avatar.current_path # => 'path/to/file.png'
u.avatar.identifier # => 'file.png'
```

### DataMapper, Mongoid, Sequel

Other ORM support has been extracted into separate gems:

* [carrierwave-datamapper](https://github.com/carrierwaveuploader/carrierwave-datamapper)
* [carrierwave-mongoid](https://github.com/carrierwaveuploader/carrierwave-mongoid)
* [carrierwave-sequel](https://github.com/jnicklas/carrierwave-sequel)

There are more extensions listed in [the wiki](https://github.com/carrierwaveuploader/carrierwave/wiki)

## Changing the storage directory

In order to change where uploaded files are put, just override the `store_dir`
method:

```ruby
class MyUploader < CarrierWave::Uploader::Base
  def store_dir
    'public/my/upload/directory'
  end
end
```

This works for the file storage as well as Amazon S3 and Rackspace Cloud Files.
Define `store_dir` as `nil` if you'd like to store files at the root level.

If you store files outside the project root folder, you may want to define `cache_dir` in the same way:

```ruby
class MyUploader < CarrierWave::Uploader::Base
  def cache_dir
    '/tmp/projectname-cache'
  end
end
```

## Securing uploads

Certain file might be dangerous if uploaded to the wrong location, such as php
files or other script files. CarrierWave allows you to specify a white-list of
allowed extensions.

If you're mounting the uploader, uploading a file with the wrong extension will
make the record invalid instead. Otherwise, an error is raised.

```ruby
class MyUploader < CarrierWave::Uploader::Base
  def extension_white_list
    %w(jpg jpeg gif png)
  end
end
```

### CVE-2016-3714 (ImageTragick)
This version of CarrierWave has the ability to mitigate CVE-2016-3714. However, you **MUST** set a `content_type_whitelist` in your uploaders for this protection to be effective, and you **MUST** either disable ImageMagick's default SVG delegate or use the RSVG delegate for SVG processing.

A valid whitelist that will restrict your uploader to images only, and mitigate the CVE is:

```ruby
class MyUploader < CarrierWave::Uploader::Base
  def content_type_whitelist
    [/image\//]
  end
end
```

**WARNING**: A `content_type_whitelist` is the only form of whitelist or blacklist supported by CarrierWave that can effectively mitigate against CVE-2016-3714. Use of `extension_white_list` will not inspect the file headers, and thus still leaves your application open to the vulnerability.


### Filenames and unicode chars

Another security issue you should care for is the file names (see
[Ruby On Rails Security Guide](http://guides.rubyonrails.org/security.html#file-uploads)).
By default, CarrierWave provides only English letters, arabic numerals and '-+_.' symbols as
white-listed characters in the file name. If you want to support local scripts (Cyrillic letters, letters with diacritics and so on), you
have to override `sanitize_regexp` method. It should return regular expression which would match
all *non*-allowed symbols.

With Ruby 1.9 and higher you can simply write (as it has [Oniguruma](http://oniguruma.rubyforge.org/oniguruma/)
built-in):

```ruby
CarrierWave::SanitizedFile.sanitize_regexp = /[^[:word:]\.\-\+]/
```

With Ruby 1.8 you have to manually specify all character ranges. For example, for files which may
contain Russian letters:

```ruby
CarrierWave::SanitizedFile.sanitize_regexp = /[^a-zA-Zа-яА-ЯёЁ0-9\.\-\+_]/u
```

Also make sure that allowing non-latin characters won't cause a compatibility issue with a third-party
plugins or client-side software.

## Setting the content type

If you care about the content type of your files and notice that it's not being set
as expected, you can configure your uploaders to use `CarrierWave::MimeTypes`.
This adds a dependency on the [mime-types](http://rubygems.org/gems/mime-types) gem,
but is recommended when using fog, and fog already has a dependency on mime-types.

```ruby
require 'carrierwave/processing/mime_types'

class MyUploader < CarrierWave::Uploader::Base
  include CarrierWave::MimeTypes

  process :set_content_type
end
```

## Adding versions

Often you'll want to add different versions of the same file. The classic example is image thumbnails. There is built in support for this*:

*Note: You must have Imagemagick and MiniMagick installed to do image resizing. MiniMagick is a Ruby interface for Imagemagick which is a C program. This is why MiniMagick fails on 'bundle install' without Imagemagick installed.

Some documentation refers to RMagick instead of MiniMagick but MiniMagick is recommended.

To install Imagemagick on OSX with homebrew type the following:

```
$ brew install imagemagick
```

```ruby
class MyUploader < CarrierWave::Uploader::Base
  include CarrierWave::MiniMagick

  process :resize_to_fit => [800, 800]

  version :thumb do
    process :resize_to_fill => [200,200]
  end

end
```

When this uploader is used, an uploaded image would be scaled to be no larger
than 800 by 800 pixels. A version called thumb is then created, which is scaled
and cropped to exactly 200 by 200 pixels. The uploader could be used like this:

```ruby
uploader = AvatarUploader.new
uploader.store!(my_file)                              # size: 1024x768

uploader.url # => '/url/to/my_file.png'               # size: 800x600
uploader.thumb.url # => '/url/to/thumb_my_file.png'   # size: 200x200
```

One important thing to remember is that process is called *before* versions are
created. This can cut down on processing cost.

It is possible to nest versions within versions:

```ruby
class MyUploader < CarrierWave::Uploader::Base

  version :animal do
    version :human
    version :monkey
    version :llama
  end
end
```

### Conditional versions

Occasionally you want to restrict the creation of versions on certain
properties within the model or based on the picture itself. Create an
instance method that accepts a file object as an argument and returns
a boolean.

```ruby
class MyUploader < CarrierWave::Uploader::Base

  version :human, :if => :is_human?
  version :monkey, :if => :is_monkey?
  version :banner, :if => :is_landscape?

protected

  def is_human? picture
    model.can_program?(:ruby)
  end

  def is_monkey? picture
    model.favorite_food == 'banana'
  end

  def is_landscape? picture
    image = MiniMagick::Image.open(picture.path)
    image[:width] > image[:height]
  end

end
```

The `model` variable points to the instance object the uploader is attached to.

### Create versions from existing versions

For performance reasons, it is often useful to create versions from existing ones
instead of using the original file. If your uploader generates several versions
where the next is smaller than the last, it will take less time to generate from
a smaller, already processed image.

```ruby
class MyUploader < CarrierWave::Uploader::Base

  version :thumb do
    process resize_to_fill: [280, 280]
  end

  version :small_thumb, :from_version => :thumb do
    process resize_to_fill: [20, 20]
  end

end
```

The option `:from_version` uses the file cached in the `:thumb` version instead
of the original version, potentially resulting in faster processing.

## Making uploads work across form redisplays

Often you'll notice that uploaded files disappear when a validation fails.
CarrierWave has a feature that makes it easy to remember the uploaded file even
in that case. Suppose your `user` model has an uploader mounted on `avatar`
file, just add a hidden field called `avatar_cache` (don't forget to add it to
the attr_accessible list as necessary). In Rails, this would look like this:

```erb
<%= form_for @user, :html => {:multipart => true} do |f| %>
  <p>
    <label>My Avatar</label>
    <%= f.file_field :avatar %>
    <%= f.hidden_field :avatar_cache %>
  </p>
<% end %>
````

It might be a good idea to show the user that a file has been uploaded, in the
case of images, a small thumbnail would be a good indicator:

```erb
<%= form_for @user, :html => {:multipart => true} do |f| %>
  <p>
    <label>My Avatar</label>
    <%= image_tag(@user.avatar_url) if @user.avatar? %>
    <%= f.file_field :avatar %>
    <%= f.hidden_field :avatar_cache %>
  </p>
<% end %>
```

## Removing uploaded files

If you want to remove a previously uploaded file on a mounted uploader, you can
easily add a checkbox to the form which will remove the file when checked.

```erb
<%= form_for @user, :html => {:multipart => true} do |f| %>
  <p>
    <label>My Avatar</label>
    <%= image_tag(@user.avatar_url) if @user.avatar? %>
    <%= f.file_field :avatar %>
  </p>

  <p>
    <label>
      <%= f.check_box :remove_avatar %>
      Remove avatar
    </label>
  </p>
<% end %>
```

If you want to remove the file manually, you can call <code>remove_avatar!</code>, then save the object.

```erb
@user.remove_avatar!
@user.save
#=> true
```

## Uploading files from a remote location

Your users may find it convenient to upload a file from a location on the Internet
via a URL. CarrierWave makes this simple, just add the appropriate attribute to your
form and you're good to go:

```erb
<%= form_for @user, :html => {:multipart => true} do |f| %>
  <p>
    <label>My Avatar URL:</label>
    <%= image_tag(@user.avatar_url) if @user.avatar? %>
    <%= f.text_field :remote_avatar_url %>
  </p>
<% end %>
```

If you're using ActiveRecord, CarrierWave will indicate invalid URLs and download
failures automatically with attribute validation errors. If you aren't, or you
disable CarrierWave's `validate_download` option, you'll need to handle those
errors yourself.

## Providing a default URL

In many cases, especially when working with images, it might be a good idea to
provide a default url, a fallback in case no file has been uploaded. You can do
this easily by overriding the `default_url` method in your uploader:

```ruby
class MyUploader < CarrierWave::Uploader::Base
  def default_url
    "/images/fallback/" + [version_name, "default.png"].compact.join('_')
  end
end
```

Or if you are using the Rails asset pipeline:

```ruby
class MyUploader < CarrierWave::Uploader::Base
  def default_url
    ActionController::Base.helpers.asset_path("fallback/" + [version_name, "default.png"].compact.join('_'))
  end
end
```

## Recreating versions

You might come to a situation where you want to retroactively change a version
or add a new one. You can use the `recreate_versions!` method to recreate the
versions from the base file. This uses a naive approach which will re-upload and
process the specified version or all versions, if none is passed as an argument.

When you are generating random unique filenames you have to call `save!` on
the model after using `recreate_versions!`. This is necessary because
`recreate_versions!` doesn't save the new filename to the database. Calling
`save!` yourself will prevent that the database and file system are running
out of sync.

```ruby
instance = MyUploader.new
instance.recreate_versions!(:thumb, :large)
```

Or on a mounted uploader:

```ruby
User.all.each do |user|
  user.avatar.recreate_versions!
end
```

Note: `recreate_versions!` will throw an exception on records without an image. To avoid this, scope the records to those with images or check if an image exists within the block. If you're using ActiveRecord, recreating versions for a user avatar might look like this:

```ruby
User.all.each do |user|
  user.avatar.recreate_versions! if user.avatar?
end
```

## Configuring CarrierWave

CarrierWave has a broad range of configuration options, which you can configure,
both globally and on a per-uploader basis:

```ruby
CarrierWave.configure do |config|
  config.permissions = 0666
  config.directory_permissions = 0777
  config.storage = :file
end
```

Or alternatively:

```ruby
class AvatarUploader < CarrierWave::Uploader::Base
  permissions 0777
end
```

If you're using Rails, create an initializer for this:

	config/initializers/carrierwave.rb


## Testing with CarrierWave

It's a good idea to test your uploaders in isolation. In order to speed up your
tests, it's recommended to switch off processing in your tests, and to use the
file storage. In Rails you could do that by adding an initializer with:

```ruby
if Rails.env.test? or Rails.env.cucumber?
  CarrierWave.configure do |config|
    config.storage = :file
    config.enable_processing = false
  end
end
```

Remember, if you have already set `storage :something` in your uploader, the `storage`
setting from this initializer will be ignored.

If you need to test your processing, you should test it in isolation, and enable
processing only for those tests that need it.

CarrierWave comes with some RSpec matchers which you may find useful:

```ruby
require 'carrierwave/test/matchers'

describe MyUploader do
  include CarrierWave::Test::Matchers

  before do
    MyUploader.enable_processing = true
    @uploader = MyUploader.new(@user, :avatar)
    @uploader.store!(File.open(path_to_file))
  end

  after do
    MyUploader.enable_processing = false
    @uploader.remove!
  end

  context 'the thumb version' do
    it "should scale down a landscape image to be exactly 64 by 64 pixels" do
      @uploader.thumb.should have_dimensions(64, 64)
    end
  end

  context 'the small version' do
    it "should scale down a landscape image to fit within 200 by 200 pixels" do
      @uploader.small.should be_no_larger_than(200, 200)
    end
  end

  it "should make the image readable only to the owner and not executable" do
    @uploader.should have_permissions(0600)
  end
end
```

Setting the enable_processing flag on an uploader will prevent any of the versions from processing as well.
Processing can be enabled for a single version by setting the processing flag on the version like so:

```ruby
@uploader.thumb.enable_processing = true
```

## Using Amazon S3

[Fog](http://github.com/fog/fog) is used to support Amazon S3. Ensure you have it in your Gemfile:

```ruby
gem "fog"
```

You'll need to provide your fog_credentials and a fog_directory (also known as a bucket) in an initializer.
For the sake of performance it is assumed that the directory already exists, so please create it if need be.
You can also pass in additional options, as documented fully in lib/carrierwave/storage/fog.rb. Here's a full example:

```ruby
CarrierWave.configure do |config|
  config.fog_credentials = {
    :provider               => 'AWS',                        # required
    :aws_access_key_id      => 'xxx',                        # required
    :aws_secret_access_key  => 'yyy',                        # required
    :region                 => 'eu-west-1',                  # optional, defaults to 'us-east-1'
    :host                   => 's3.example.com',             # optional, defaults to nil
    :endpoint               => 'https://s3.example.com:8080' # optional, defaults to nil
  }
  config.fog_directory  = 'name_of_directory'                     # required
  config.fog_public     = false                                   # optional, defaults to true
  config.fog_attributes = {'Cache-Control'=>'max-age=315576000'}  # optional, defaults to {}
end
```

In your uploader, set the storage to :fog

```ruby
class AvatarUploader < CarrierWave::Uploader::Base
  storage :fog
end
```

That's it! You can still use the `CarrierWave::Uploader#url` method to return the url to the file on Amazon S3.

## Using Rackspace Cloud Files

[Fog](http://github.com/fog/fog) is used to support Rackspace Cloud Files. Ensure you have it in your Gemfile:

```ruby
gem "fog"
```

You'll need to configure a directory (also known as a container), username and API key in the initializer.
For the sake of performance it is assumed that the directory already exists, so please create it if need be.

Using a US-based account:

```ruby
CarrierWave.configure do |config|
  config.fog_credentials = {
    :provider           => 'Rackspace',
    :rackspace_username => 'xxxxxx',
    :rackspace_api_key  => 'yyyyyy',
    :rackspace_region   => :ord                # optional, defaults to :dfw
  }
  config.fog_directory = 'name_of_directory'
end
```

Using a UK-based account:

```ruby
CarrierWave.configure do |config|
  config.fog_credentials = {
    :provider           => 'Rackspace',
    :rackspace_username => 'xxxxxx',
    :rackspace_api_key  => 'yyyyyy',
    :rackspace_auth_url  => Fog::Rackspace::UK_AUTH_ENDPOINT,
    :rackspace_region   => :lon
  }
  config.fog_directory = 'name_of_directory'
end
```

You can optionally include your CDN host name in the configuration.
This is *highly* recommended, as without it every request requires a lookup
of this information.

```ruby
config.asset_host = "http://c000000.cdn.rackspacecloud.com"
```

In your uploader, set the storage to :fog

```ruby
class AvatarUploader < CarrierWave::Uploader::Base
  storage :fog
end
```

That's it! You can still use the `CarrierWave::Uploader#url` method to return
the url to the file on Rackspace Cloud Files.

## Using Google Storage for Developers

[Fog](http://github.com/fog/fog) is used to support Google Storage for Developers. Ensure you have it in your Gemfile:

```ruby
gem "fog"
```

You'll need to configure a directory (also known as a bucket), access key id and secret access key in the initializer.
For the sake of performance it is assumed that the directory already exists, so please create it if need be.

Sign up [here](http://gs-signup-redirect.appspot.com/) and get your credentials [here](https://storage.cloud.google.com/m)
under the section “Interoperable Access”.


```ruby
CarrierWave.configure do |config|
  config.fog_credentials = {
    :provider                         => 'Google',
    :google_storage_access_key_id     => 'xxxxxx',
    :google_storage_secret_access_key => 'yyyyyy'
  }
  config.fog_directory = 'name_of_directory'
end
```

In your uploader, set the storage to :fog

```ruby
class AvatarUploader < CarrierWave::Uploader::Base
  storage :fog
end
```

That's it! You can still use the `CarrierWave::Uploader#url` method to return
the url to the file on Google.

## Dynamic Asset Host

The `asset_host` config property can be assigned a proc (or anything that responds to `call`) for generating the host dynamically. The proc-compliant object gets an instance of the current `CarrierWave::Storage::Fog::File` or `CarrierWave::SanitizedFile` as its only argument.

```ruby
CarrierWave.configure do |config|
  config.asset_host = proc do |file|
    identifier = # some logic
    "http://#{identifier}.cdn.rackspacecloud.com"
  end
end
```

## Using RMagick

If you're uploading images, you'll probably want to manipulate them in some way,
you might want to create thumbnail images for example. CarrierWave comes with a
small library to make manipulating images with RMagick easier, you'll need to
include it in your Uploader:

```ruby
class AvatarUploader < CarrierWave::Uploader::Base
  include CarrierWave::RMagick
end
```

The RMagick module gives you a few methods, like
`CarrierWave::RMagick#resize_to_fill` which manipulate the image file in some
way. You can set a `process` callback, which will call that method any time a
file is uploaded.
There is a demonstration of convert here.
Convert will only work if the file has the same file extension, thus the use of the filename method.

```ruby
class AvatarUploader < CarrierWave::Uploader::Base
  include CarrierWave::RMagick

  process :resize_to_fill => [200, 200]
  process :convert => 'png'

  def filename
    super.chomp(File.extname(super)) + '.png' if original_filename.present?
  end
end
```

Check out the manipulate! method, which makes it easy for you to write your own
manipulation methods.

## Using MiniMagick

MiniMagick is similar to RMagick but performs all the operations using the 'mogrify'
command which is part of the standard ImageMagick kit. This allows you to have the power
of ImageMagick without having to worry about installing all the RMagick libraries.

See the MiniMagick site for more details:

https://github.com/minimagick/minimagick

And the ImageMagick command line options for more for whats on offer:

http://www.imagemagick.org/script/command-line-options.php

Currently, the MiniMagick carrierwave processor provides exactly the same methods as
for the RMagick processor.

```ruby
class AvatarUploader < CarrierWave::Uploader::Base
  include CarrierWave::MiniMagick

  process :resize_to_fill => [200, 200]
end
```

## Migrating from Paperclip

If you are using Paperclip, you can use the provided compatibility module:

```ruby
class AvatarUploader < CarrierWave::Uploader::Base
  include CarrierWave::Compatibility::Paperclip
end
```

See the documentation for `CarrierWave::Compatibility::Paperclip` for more
details.

Be sure to use mount_on to specify the correct column:

```ruby
mount_uploader :avatar, AvatarUploader, :mount_on => :avatar_file_name
```

Unfortunately attachment_fu differs too much in philosophy for there to be a
sensible compatibility mode. Patches for migrating from other solutions will be
happily accepted.

## i18n

The Active Record validations use the Rails i18n framework. Add these keys to
your translations file:

```yaml
errors:
  messages:
    carrierwave_processing_error: "Cannot resize image."
    carrierwave_integrity_error: "Not an image."
    carrierwave_download_error: "Couldn't download image."
    extension_white_list_error: "You are not allowed to upload %{extension} files, allowed types: %{allowed_types}"
    extension_black_list_error: "You are not allowed to upload %{extension} files, prohibited types: %{prohibited_types}"
```

## Large files

By default, CarrierWave copies an uploaded file twice, first copying the file into the cache, then
copying the file into the store.  For large files, this can be prohibitively time consuming.

You may change this behavior by overriding either or both of the `move_to_cache` and
`move_to_store` methods:

```ruby
class MyUploader < CarrierWave::Uploader::Base
  def move_to_cache
    true
  end
  def move_to_store
    true
  end
end
```

When the `move_to_cache` and/or `move_to_store` methods return true, files will be moved (instead of copied) to the cache and store respectively.

This has only been tested with the local filesystem store.

## Skipping ActiveRecord callbacks

By default, mounting an uploader into an ActiveRecord model will add a few
callbacks. For example, this code:

```ruby
class User
  mount_uploader :avatar, AvatarUploader
end
```

Will add these callbacks:

```ruby
after_save :store_avatar!
before_save :write_avatar_identifier
after_commit :remove_avatar! :on => :destroy
before_update :store_previous_model_for_avatar
after_save :remove_previously_stored_avatar
```

If you want to skip any of these callbacks (eg. you want to keep the existing
avatar, even after uploading a new one), you can use ActiveRecord’s
`skip_callback` method.

```ruby
class User
  mount_uploader :avatar, AvatarUploader
  skip_callback :save, :after, :remove_previously_stored_avatar
end
```

## Contributing to CarrierWave

See [CONTRIBUTING.md](https://github.com/carrierwaveuploader/carrierwave/blob/master/CONTRIBUTING.md)

## License

Copyright (c) 2008-2013 Jonas Nicklas

Permission is hereby granted, free of charge, to any person obtaining
a copy of this software and associated documentation files (the
"Software"), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE
LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION
OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION
WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
