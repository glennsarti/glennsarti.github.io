is_windows = (ENV['OS'] == 'Windows_NT')

# clean settings
desc 'Clean'
task :clean do
  puts 'Cleaning Jekyll'
  system 'jekyll clean'
end

basicSettings = '' #'--trace --safe'
devConfig = '--config _config.yml,_config.dev.yml --unpublished --future'

namespace :build do
  desc 'Build Jekyll with production settings'
  task :prod => [:clean] do
    puts 'Building Jekyll with PRODUCTION settings...'
    system "JEKYLL_ENV=production jekyll build #{basicSettings} --no-watch"
  end

  desc 'Build Jekyll with development settings'
  task :dev do
    puts 'Building Jekyll with DEVELOPMENT settings...'
    system "JEKYLL_ENV=development jekyll build #{basicSettings} #{devConfig} --no-watch"
  end
end

namespace :serve do
  desc 'Serve Jekyll with production settings'
  task :prod => [:clean] do
    puts 'Building Jekyll with PRODUCTION settings...'
    if is_windows then
      ENV['JEKYLL_ENV'] = "production"
      system "jekyll serve #{basicSettings}"
    else
      system "JEKYLL_ENV=production jekyll serve #{basicSettings}"
    end
  end

  desc 'Serve Jekyll with development settings'
  task :dev => [:clean] do
    puts 'Building Jekyll with DEVELOPMENT settings...'
    if is_windows then
      ENV['JEKYLL_ENV'] = "development"
      system "jekyll serve #{basicSettings} #{devConfig}"
    else
      system "JEKYLL_ENV=development jekyll serve #{basicSettings} #{devConfig}"
    end
  end

end

namespace :watch do
  desc 'Serve and watch Jekyll with development settings'
  task :dev => [:clean] do
    puts 'Building Jekyll with DEVELOPMENT settings...'
    if is_windows then
      ENV['JEKYLL_ENV'] = "development"
      system "jekyll serve #{basicSettings} #{devConfig} --watch --incremental"
    else
      system "JEKYLL_ENV=development jekyll serve --incremental #{basicSettings} #{devConfig} --watch"
    end
  end
end
