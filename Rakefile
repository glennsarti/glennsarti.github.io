thisdir = File.dirname(__FILE__)
maincss_file = File.expand_path("#{thisdir}/assets/css/main.css")
maincss_file_site = File.expand_path("#{thisdir}/_site/assets/css/main.css")
mainscss_file = File.expand_path("#{thisdir}/assets/css/main.scss")
mainscss_file_content = "---\n# Front matter comment to ensure Jekyll properly reads file.\n---\n\n@import \"main\";"

def write_file(filename,content)
  File.open(filename, "w:UTF-8") { |file| file.write(content) }
end

# clean settings
desc 'Clean'
task :clean do
  puts 'Cleaning Jekyll'
  system 'jekyll clean'
  
  if File.exists?(maincss_file)
    File.delete(maincss_file)
  end
  if !File.exists?(mainscss_file)
    write_file(mainscss_file,mainscss_file_content)
  end
end

basicSettings = '--trace --no-watch --safe'
devConfig = '--config _config.yml,_config.dev.yml'

namespace :build do

  desc 'Build Jekyll with production settings'
  task :prod => [:clean] do
    puts 'Building Jekyll with PRODUCTION settings...'
    system "JEKYLL_ENV=production jekyll build #{basicSettings}"
  end

  desc 'Build Jekyll with development settings'
  task :dev do
    puts 'Building Jekyll with DEVELOPMENT settings...'
    
    # Disable scss compilation if needed
    if File.exists?(maincss_file) 
      if File.exists?(mainscss_file)
        File.delete(mainscss_file)
      end
    else
      if !File.exists?(mainscss_file)
        write_file(mainscss_file,mainscss_file_content)
      end
    end

    system "JEKYLL_ENV=development jekyll build #{basicSettings} #{devConfig}"
    
    # Cache the main.css to speed up compilation
    if !File.exists?(maincss_file)
      FileUtils.cp(maincss_file_site,maincss_file) 
    end
  end

end

namespace :serve do

  desc 'Serve Jekyll with production settings'
  task :prod => [:clean] do
    puts 'Building Jekyll with PRODUCTION settings...'
    system "JEKYLL_ENV=production jekyll serve #{basicSettings}"
  end

  desc 'Serve Jekyll with development settings'
  task :dev => [:clean] do
    puts 'Building Jekyll with DEVELOPMENT settings...'
    system "JEKYLL_ENV=development jekyll serve #{basicSettings} #{devConfig}"
  end

end

namespace :watch do
  desc 'Serve and watch Jekyll with development settings'
  task :dev => [:clean] do
    puts 'Building Jekyll with DEVELOPMENT settings...'
    system "JEKYLL_ENV=development jekyll serve #{basicSettings} #{devConfig} --watch"
  end

end
