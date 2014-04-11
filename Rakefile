require 'tmpdir'
require 'fileutils'

task :default => [:test]

desc "Compile the site"
task :compile do
  `nanoc compile`
end

desc "Test the output"
task :test => [:remove_output_dir, :compile] do
  require 'html/proofer'
  HTML::Proofer.new("./output").run
end

desc "Remove the output dir"
task :remove_output_dir do
  FileUtils.rm_r('output') if File.exist?('output')
end

# Prompt user for a commit message; default: P U B L I S H :emoji:
def commit_message
  publish_emojis = [':boom:', ':rocket:', ':metal:', ':bulb:', ':zap:',
    ':sailboat:', ':gift:', ':ship:', ':shipit:', ':sparkles:', ':rainbow:']
  default_message = "P U B L I S H #{publish_emojis.sample}"

  default_message.gsub(/'/, '') # Allow this to be handed off via -m '#{message}'
end

desc "Publish to http://developer.github.com"
task :publish => [:remove_output_dir] do
  mesg = commit_message

  sh "nanoc compile"

  Dir.mktmpdir do |tmp|
    system "mv output/* #{tmp}"
    system "git checkout gh-pages"
    system "rm -rf *"
    system "mv #{tmp}/* ."
    system "git add ."
    system "git commit -am #{mesg.shellescape}"
    system "git push origin gh-pages --force"
    system "git checkout master"
    system "echo yolo"
  end
end
