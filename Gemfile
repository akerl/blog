path = File.dirname(ENV['BUNDLE_GEMFILE'] || '.')
ruby File.read(File.join(path, '.circle-ruby')).chomp
source 'https://rubygems.org'

gem 'jekyll', '~> 3.5.0'
gem 'jekyll-gist', '~> 1.4.0'
gem 'kramdown', '~> 1.15.0'
gem 'jekyll-assets', '~> 2.3.0'
gem 'uglifier', '~> 3.2.0'
gem 'sass', '~> 3.5.0'
gem 'therubyracer', '~> 0.12.3'
gem 'html-proofer', '~> 3.7.0'

group :development do
  gem 'rubocop', '~> 0.49.0'
  gem 'rake', '~> 12.0.0'
  gem 'rspec', '~> 3.6.0'
  gem 'fuubar', '~> 2.2.0'
end
