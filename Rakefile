# clean settings
desc 'Clean'
task :clean do
  puts 'Cleaning Jekyll'
  system 'jekyll clean'
end

basicSettings = '--trace --no-watch --safe'
devConfig = '--config _config.yml,_config.dev.yml'

namespace :build do

  desc 'Build Jekyll with production settings'
  task :prod do
    puts 'Building Jekyll with PRODUCTION settings...'
    system "JEKYLL_ENV=production jekyll build #{basicSettings}"
  end

  desc 'Build Jekyll with development settings'
  task :dev do
    puts 'Building Jekyll with DEVELOPMENT settings...'
    system "JEKYLL_ENV=development jekyll build #{basicSettings} #{devConfig}"
  end

  # desc 'Build CSS with Jekyll'
  # task :css do
  #   puts 'Building Jekyll with DEVELOPMENT settings...'
  #   system "JEKYLL_ENV=development jekyll build ${basicSettings} ${devConfig}"
  # end

end

namespace :serve do

  desc 'Serve Jekyll with production settings'
  task :prod do
    puts 'Building Jekyll with PRODUCTION settings...'
    system "JEKYLL_ENV=production jekyll serve #{basicSettings}"
  end

  desc 'Serve Jekyll with development settings'
  task :dev do
    puts 'Building Jekyll with DEVELOPMENT settings...'
    system "JEKYLL_ENV=development jekyll serve #{basicSettings} #{devConfig}"
  end

end

# # build settings
# desc 'Build Jekyll with production settings'
# task :build do
#   puts 'Building Jekyll with PRODUCTION settings...'
#   system 'JEKYLL_ENV=production jekyll build --trace --no-watch --safe'
# end

# # build settings
# desc 'Build Jekyll with development settings'
# task :build_dev do
#   puts 'Building Jekyll with DEVELOPMENT settings...'
#   system 'JEKYLL_ENV=development jekyll build --trace --no-watch --safe --config _config.yml,_config.dev.yml'
# end

# # serve settings
# desc 'Run Jekyll with production settings'
# task :serve do
#   puts 'Serving Jekyll with PRODUCTION settings...'
#   system 'JEKYLL_ENV=production jekyll serve --trace --no-watch --safe'
# end

# # serve settings
# desc 'Run Jekyll with development settings'
# task :serve_dev do
#   puts 'Serving Jekyll with DEVELOPMENT settings...'
#   system 'JEKYLL_ENV=development jekyll serve --trace --no-watch --safe --config _config.yml,_config.dev.yml'
# end
