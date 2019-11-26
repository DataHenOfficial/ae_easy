[![Gem Version](https://badge.fury.io/rb/dh_easy.svg)](http://github.com/DataHenOfficial/dh_easy/releases)
[![License](http://img.shields.io/badge/license-MIT-yellowgreen.svg)](#license)

# DhEasy
## Description

DhEasy gem collection allow advance DataHen features possible by including a collection of specialized gems.

Install gem:
```ruby
gem install 'dh_easy'
```

Require gem:
```ruby
require 'dh_easy'
```

Included gems documentation:
```
dh_easy-core: http://rubydoc.org/gems/dh_easy-core/frames
dh_easy-config: http://rubydoc.org/gems/dh_easy-config/frames
dh_easy-router: http://rubydoc.org/gems/dh_easy-router/frames
dh_easy-text: http://rubydoc.org/gems/dh_easy-text/frames
dh_easy-login: http://rubydoc.org/gems/dh_easy-login/frames
```

## How to implement

### Sample DataHen project

Lets take a simple project without `dh_easy`:

```yaml
# ./config.yaml

seeder:
  file: ./seeder/seeder.rb
  disabled: false
parsers:
  - page_type: search
    file: ./parsers/search.rb
    disabled: false
  - page_type: product
    file: ./parsers/product.rb
    disabled: false
```

```ruby
# ./seeder/seeder.rb

pages << {
  'url' => 'https://example.com/login.rb?query=food',
  'page_type' => 'search'
}
```

```ruby
# ./parsers/search.rb

require 'cgi'

html = Nokogiri.HTML content
html.css('.name').each do |element|
  name = element.text.strip
  pages << {
    'url' => "https://example.com/product/#{CGI::escape name}",
    'page_type' => 'product',
    'vars' => {'name' => name}
  }
end
```

```ruby
# ./parsers/product.rb

html = Nokogiri.HTML content
description = html.css('.description').first.text.strip
outputs << {
  '_collection' => 'product',
  'name' => page['vars']['name'],
  'description' => description
}
```

### Adding dh_easy to sample project

One of DhEasy's main feature is to allow users to use classes instead of raw scripts with the whole `datahen` gem contexts (seeder, parsers, finishers, etc.) functions and objects integreated directly on our classes.

Converting seeders, parsers and finishers to DhEasy supported classes is quite easy, just wrap your seeders and parsers like this:

```ruby
class MySeeder
  include DhEasy::Core::Plugin::Seeder

  # Create "initialize_hook_*" methods instead of "initialize" method
  #  to prevent overriding the logic behind DhEasy
  def initialize_hook_my_seeder opts = {}
    @my_param = opts[:my_param]
  end

  def seed

    # Your seeder code goes here

  end
end
```

```ruby
class MyParser
  include DhEasy::Core::Plugin::Parser

  # Create "initialize_hook_*" methods instead of "initialize" method
  #  to prevent overriding the logic behind DhEasy
  def initialize_hook_my_parser opts = {}
    @my_param = opts[:my_param]
  end

  def parse

    # Your parser code goes here

  end
end
```

```ruby
class MyFinisher
  include DhEasy::Core::Plugin::Finisher

  # Create "initialize_hook_*" methods instead of "initialize" method
  #  to prevent overriding the logic behind DhEasy
  def initialize_hook_my_parser opts = {}
    @my_param = opts[:my_param]
  end

  def finish

    # Your finisher code goes here

  end
end
```

You can also add `initialize_hook_` methods to extend the default `initialize` provided by DhEasy plugins.

Now let's try this on our sample project's seeders and parsers:

```ruby
# ./seeder/seeder.rb

module Seeder
  class Seeder
    include DhEasy::Core::Plugin::Seeder

    def seed
      pages << {
        'url' => 'https://example.com/search?query=food',
        'page_type' => 'search'
      }
    end
  end
end
```

```ruby
# ./parsers/search.rb

module Parsers
  class Search
    include DhEasy::Core::Plugin::Parser

    def parse
      html = Nokogiri.HTML content
      html.css('.name').each do |element|
        name = element.text.strip
        pages << {
          'url' => "https://example.com/product/#{CGI::escape name}",
          'page_type' => 'product',
          'vars' => {'name' => name}
        }
      end
    end
  end
end
```

```ruby
# ./parsers/product.rb

module Parsers
  class Product
    include DhEasy::Core::Plugin::Parser

    def parse
      html = Nokogiri.HTML content
      description = html.css('.description').first.text.strip
      outputs << {
        '_collection' => 'product',
        'name' => page['vars']['name'],
        'description' => description
      }
    end
  end
end
```

Next step is to add router capabilities to consume these classes. To do this, let's create the routers and require our seeder and parsers classes, like this:

```ruby
# ./router/seeder.rb

require 'dh_easy/router'
require './seeder/seeder'

DhEasy::Router::Seeder.new.route context: self
```

```ruby
# ./router/parser.rb

require 'cgi'
require 'dh_easy/router'
require './parsers/search'
require './parsers/product'

DhEasy::Router::Parser.new.route context: self
```

Now lets create our `./dh_easy.yaml` config file to link our routers to our new seeder and parsers classes:

```yaml
# ./dh_easy.yaml

router:
  parser:
    routes:
      - page_type: search
        class: Parsers::Search
      - page_type: product
        class: Parsers::Product

  seeder:
    routes:
      - class: Seeder::Seeder
```

Finally, we need to modify our `./config.yaml` to use our routers:

```yaml
# ./config.yaml

seeder:
  file: ./router/seeder.rb
  disabled: false

parsers:
  - page_type: search
    file: ./router/parser.rb
    disabled: false
  - page_type: product
    file: ./router/parser.rb
    disabled: false
```

Hurray! you have successfullly implemented DhEasy on your project.
