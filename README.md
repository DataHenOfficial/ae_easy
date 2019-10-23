[![Gem Version](https://badge.fury.io/rb/ae_easy.svg)](http://github.com/answersengine/ae_easy/releases)
[![License](http://img.shields.io/badge/license-MIT-yellowgreen.svg)](#license)

# AeEasy
## Description

AeEasy gem collection allow advance AnswersEngine features possible by including a collection of specialized gems.

Install gem:
```ruby
gem install 'ae_easy'
```

Require gem:
```ruby
require 'ae_easy'
```

Included gems documentation:
```
ae_easy-core: http://rubydoc.org/gems/ae_easy-core/frames
ae_easy-config: http://rubydoc.org/gems/ae_easy-config/frames
ae_easy-router: http://rubydoc.org/gems/ae_easy-router/frames
ae_easy-text: http://rubydoc.org/gems/ae_easy-text/frames
ae_easy-login: http://rubydoc.org/gems/ae_easy-login/frames
```

## How to implement

### Sample AnswersEngine project

Lets take a simple project without `ae_easy`:

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

### Adding ae_easy to sample project

One of AeEasy's main feature is to allow users to use classes instead of raw scripts with the whole `answersengine` gem contexts (seeder, parsers, finishers, etc.) functions and objects integreated directly on our classes.

Converting seeders, parsers and finishers to AeEasy supported classes is quite easy, just wrap your seeders and parsers like this:

```ruby
class MySeeder
  include AeEasy::Core::Plugin::Seeder
  
  # Create "initialize_hook_*" methods instead of "initialize" method
  #  to prevent overriding the logic behind AeEasy
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
  include AeEasy::Core::Plugin::Parser
  
  # Create "initialize_hook_*" methods instead of "initialize" method
  #  to prevent overriding the logic behind AeEasy
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
  include AeEasy::Core::Plugin::Finisher
  
  # Create "initialize_hook_*" methods instead of "initialize" method
  #  to prevent overriding the logic behind AeEasy
  def initialize_hook_my_parser opts = {}
    @my_param = opts[:my_param]
  end
  
  def finish

    # Your finisher code goes here

  end
end
```

You can also add `initialize_hook_` methods to extend the default `initialize` provided by AeEasy plugins.

Now let's try this on our sample project's seeders and parsers:

```ruby
# ./seeder/seeder.rb

module Seeders
  class Seeder
    include AeEasy::Core::Plugin::Seeder

    def seed
      pages << {
        'url' => 'https://example.com/login.rb?query=food',
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

require 'ae_easy/router'
require './seeder/seeder.rb'

AeEasy::Router::Seeder.new.route context: self
```

```ruby
# ./router/parser.rb

require 'cgi'
require 'ae_easy/router'
require './parsers/search.rb'
require './parsers/product.rb'

AeEasy::Router::Parser.new.route context: self
```

Now lets create our `./ae_easy.yaml` config file to link our routers to our new seeder and parsers classes:

```yaml
# ./ae_easy.yaml

router:
  parser:
    routes:
      - page_type: search
        class: Parsers::Search
      - page_type: product
        class: Parsers::Product
  seeder:
    routes:
      - class: Seeders::Search
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

Hurray! you have successfullly implemented AeEasy on your project.
