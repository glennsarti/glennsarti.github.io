source ENV['GEM_SOURCE'] || "https://rubygems.org"

gem "rake"

if Gem.win_platform?
  # Jekyll really hates Windows :-(  Need to use WSL instead
  raise 'Jekyll really hates Windows :-(  Need to use WSL instead'
else
  gem "jekyll"
  gem "jemoji"
  # Remember to update the _config.yml with the same version
  gem "minimal-mistakes-jekyll", "4.19.3"
end

# Evaluate Gemfile.local if it exists
if File.exists? "#{__FILE__}.local"
  eval(File.read("#{__FILE__}.local"), binding)
end