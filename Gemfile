path = File.dirname(ENV['BUNDLE_GEMFILE'] || '.')
ruby File.read(File.join(path, '.circle-ruby')).chomp
source 'https://rubygems.org'

gem 'html-proofer', '~> 3.13.0'
gem 'jekyll', '~> 3.8.0'
gem 'therubyracer', '~> 0.12.3'
gem 'uglifier', '~> 4.2.0'

group :jekyll_plugins do
  gem 'jekyll-assets', '~> 3.0.1'
  gem 'jekyll-gist', '~> 1.5.0'
end

group :development do
  gem 'goodcop', '~> 0.8.0'
end
