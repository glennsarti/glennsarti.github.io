source ENV['GEM_SOURCE'] || "https://rubygems.org"

gem "github-pages"
gem "rake"
# gem "jekyll-archives"
gem "wdm", "~> 0.1.0" if Gem.win_platform?

# Evaluate Gemfile.local if it exists
if File.exists? "#{__FILE__}.local"
  eval(File.read("#{__FILE__}.local"), binding)
end