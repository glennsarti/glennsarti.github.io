source ENV['GEM_SOURCE'] || "https://rubygems.org"

gem "rake"

if Gem.win_platform?
  # Jekyll 3.2.1 allows --watch on Windows, future versions don't  :sad:
  gem "jekyll", "=3.2.1"
  gem "github-pages", "101"

  gem "wdm", "~> 0.1.0"
else
  # WSL or native non-Windows
  gem "github-pages", group: :jekyll_plugins
  
  group :jekyll_plugins do
    gem "jekyll-paginate"
    gem "jekyll-sitemap"
    gem "jekyll-gist"
    gem "jekyll-feed"
    gem "jemoji"
  end
end

# Evaluate Gemfile.local if it exists
if File.exists? "#{__FILE__}.local"
  eval(File.read("#{__FILE__}.local"), binding)
end