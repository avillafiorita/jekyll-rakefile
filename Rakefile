task :default => :preview

# CONFIGURATION VARIABLES
#
# MANAGING DEPLOYMENT.
# These variables specify where to deploy your website:
#
# $deploy_url = "http://www.example.com/somedir"    # where the system will live
# $deploy_dir = "user@host:~/some-location/"        # where the sources live
#
# PREVIEWING
# If your project is based on compass and you want compass to be invoked
# by the script, set the $compass variable to true
#
# $compass = false
#
# MANAGING POSTS
#
# Extension for new posts (defaults to .textile, if not set)
#
# $post_ext = ".textile"  # default
# $post_ext = ".md"       # if you prefer markdown
#
# Location of new posts (defaults to "_posts/", if not set).
# Please, terminate with a slash:
#
# $post_dir = "_posts/"
#
# ... or load them from a file, e.g.:
#
load '_rake-configuration.rb'

#
# --- NO NEED TO TOUCH ANYTHING BELOW THIS LINE ---
#

# Specify default values for variables not set by the user

$post_ext ||= ".textile"
$post_dir ||= "_posts/"

#
# Tasks start here
#

desc 'Clean up generated site'
task :clean do
  cleanup
end

desc 'Preview on local machine (server with --auto)'
task :preview => :clean do
  set_url('http://localhost:4000')
  compass('compile') # so that we are sure sass has been compiled before we run the server
  compass('watch &')
  jekyll('serve --watch')
end
task :serve => :preview

desc 'Static build (build using filesystem)'
task :build_static => :clean do
  set_url(Dir.getwd + "/_site")
  compass('compile')
  jekyll('build')
end

desc 'Build for deployment (but do not deploy)'
task :build => :clean do
  if rake_running then
    puts "\n\nWarning! An instance of rake seems to be running (it might not be *this* Rakefile, however).\n"
    puts "Building while running other tasks (e.g., preview), might create a website with broken links.\n\n"
    puts "Are you sure you want to continue? [Y|n]"

    ans = STDIN.gets.chomp
    break if ans != 'Y' 
  end

  set_url($deploy_url)
  compass('compile')
  jekyll('build')
end

desc 'Build and deploy to remote server'
task :deploy => :build do
  sh "rsync -avz --delete _site/ #{$deploy_dir}"
  File.open("_last_deploy.txt", 'w') {|f| f.write(Time.new) }
end

desc 'Create a post'
task :create_post, [:title, :date, :category, :content] do |t, args|
  if args.title == nil then
    puts "Error! title is empty"
    puts "Usage: create_post[title,date,category,content]"
    puts "DATE and CATEGORY are optional"
    exit 1
  end
  if (args.date != nil and args.date.match(/[0-9]+-[0-9]+-[0-9]+/) == nil) then
    puts "Error: date not understood"
    puts "Usage: create_post[title,date,category,content]"
    puts "DATE and CATEGORY are optional"
    exit 1
  end

  post_title= args.title
  post_date= args.date || Time.new.strftime("%Y-%m-%d %H:%M:%S %Z")

  # the destination directory is <<category>>/$post_dir, if category is non-nil
  # and the directory exists; $post_dir otherwise (a category tag is added in
  # the post body, in this case)
  category = args.category
  if category and Dir.exists?(category + "/" + $post_dir) then
    post_dir = category + "/" + $post_dir
    yaml_cat = nil
  else
    post_dir = $post_dir
    yaml_cat = "category: #{category}\n"
  end

  def slugify (title)
    # strip characters and whitespace to create valid filenames, also lowercase
    return title.downcase.strip.gsub(' ', '-').gsub(/[^\w-]/, '')
  end

  filename = post_date[0..9] +"-"+ slugify( post_title ) + $post_ext

  # generate a unique filename appending a number
  i = 1
  while File.exists?(post_dir + filename) do
    filename = post_date[0..9] + "-" + 
               post_title.gsub(' ', '_') + "-" + i.to_s + 
               ".textile"
    i += 1
  end

  # the condition is not really necessary anymore (since the previous
  # loop ensures the file does not exist)
  if not File.exists?(post_dir + filename) then
      File.open(post_dir + filename, 'w') do |f|
        f.puts "---"
        f.puts "title: \"#{post_title}\""
        f.puts "layout: default"
        f.puts yaml_cat if yaml_cat != nil
        f.puts "date: #{post_date}"
        f.puts "---"
        f.puts args.content if args.content != nil
      end  

      puts "Post created under \"#{post_dir}#{filename}\""

      sh "open \"#{post_dir}#{filename}\"" if args.content == nil
  else
    puts "A post with the same name already exists. Aborted."
  end
  # puts "You might want to: edit #{$post_dir}#{filename}"
end

desc 'Create a post with all changes since last deploy'
task :post_changes do |t, args|
  content = "Recent changes on the website:\n\n"
  content << "<ul>\n"
  IO.popen('find * -newer _last_deploy.txt') do |io| 
    while (line = io.gets) do
      content << myprocess(line)
    end
  end 
  content << "</ul>\n"

  Rake::Task["create_post"].invoke("Recent Changes", Time.new.strftime("%Y-%m-%d %H:%M:%S"), nil, content)
end

#
# support functions for generating list of changed files
#
def myprocess(filename)
  new_filename = filename[0, filename.length - 1]
  if file_matches(new_filename) then
    puts new_filename
    "<li><a href=\"{{site.url}}/#{file_change_ext(new_filename, ".html")}\">#{new_filename}</a></li>\n"
  else
    ""
  end
end

EXCLUSION_REGEXPS = [/.*~/, /_.*/, /javascripts/, /stylesheets/, /Rakefile/, /Gemfile/, /s[ca]ss/, /.*\.css/, /.*.js/]

def file_matches(filename)
  output = EXCLUSION_REGEXPS.each.collect { |x| filename.index(x) != nil }.include?(true)
  not output
end 

def file_change_ext(filename, newext)
  if File.extname(filename) == ".textile" or File.extname(filename) == ".md" then
    filename.sub(File.extname(filename), newext)
  else  
    filename
  end
end

desc 'Check links for site already running on localhost:4000'
task :check_links do
  begin
    require 'anemone'

    root = 'http://localhost:4000/'
    puts "Checking links with anemone ... "
    # check-links --no-warnings http://localhost:4000
    Anemone.crawl(root, :discard_page_bodies => true) do |anemone|
      anemone.after_crawl do |pagestore|
        broken_links = Hash.new { |h, k| h[k] = [] }
        pagestore.each_value do |page|
          if page.code != 200
            referrers = pagestore.pages_linking_to(page.url)
            referrers.each do |referrer|
              broken_links[referrer] << page
            end
          else
            puts "OK #{page.url}"
          end
        end
        puts "\n\nLinks with issues: "
        broken_links.each do |referrer, pages|
          puts "#{referrer.url} contains the following broken links:"
          pages.each do |page|
            puts "  HTTP #{page.code} #{page.url}"
          end
        end
      end
    end
    puts "... done!"

  rescue LoadError
    abort 'Install anemone gem: gem install anemone'
  end
end

def cleanup
  sh 'rm -rf _site'
  compass('clean')
end

def jekyll(directives = '')
  sh 'jekyll ' + directives
end

def compass(command = 'compile')
  (sh 'compass ' + command) if $compass
end

# check if there is another rake task running
# (in addition to this one!)
def rake_running
  `ps | grep 'rake' | grep -v 'grep' | wc -l`.to_i > 1
end

# set the url in the configuration file
def set_url(url)
  if $deploy_url != nil
    config_filename = "_config.yml"
    text = File.read(config_filename)
    url_directive = Regexp.new(/^url: .*$/)
    if text.match(url_directive)
      puts = text.gsub(url_directive, "url: #{url}")
    else
      puts = text + "\nurl: #{url}"
    end
    File.open(config_filename, "w") { |file| file << puts }
  end
end 

