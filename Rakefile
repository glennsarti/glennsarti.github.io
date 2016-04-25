# clean settings
desc 'Clean'
task :clean do
  puts 'Cleaning Jekyll'
  system 'jekyll clean'
end

# build settings
desc 'Build Jekyll with production settings'
task :build do
  puts 'Building Jekyll with PRODUCTION settings...'
  system 'JEKYLL_ENV=production jekyll build --no-watch'
end

# build settings
desc 'Build Jekyll with development settings'
task :build_dev do
  puts 'Building Jekyll with DEVELOPMENT settings...'
  system 'JEKYLL_ENV=development jekyll build --no-watch --config _config.yml,_config.dev.yml'
end

# serve settings
desc 'Run Jekyll with production settings'
task :serve do
  puts 'Serving Jekyll with PRODUCTION settings...'
  system 'JEKYLL_ENV=production jekyll serve --no-watch'
end

# serve settings
desc 'Run Jekyll with development settings'
task :serve_dev do
  puts 'Serving Jekyll with DEVELOPMENT settings...'
  system 'JEKYLL_ENV=development jekyll serve --no-watch --config _config.yml,_config.dev.yml'
end
