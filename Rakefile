# Adapted from Remi Prevost’s Rakefiles
# http://github.com/remiprev/notes.remiprevost.com/blob/master/Rakefile

task :default => :server

desc 'Start server with --auto'
task :server do
  jekyll('--server --auto')
end

desc 'Build site with Jekyll'
task :build do
  jekyll('--no-future')
end

desc 'Build and deploy'
task :publish => :build do
  bucket = 'jmlacroix.com'
  puts "Publishing site to bucket #{bucket}"
  sh 'ruby aws_cf_sync.rb _site/ ' + bucket
end

def jekyll(opts = '')
  sh 'rm -rf _site/*'
  sh 'jekyll ' + opts
end
