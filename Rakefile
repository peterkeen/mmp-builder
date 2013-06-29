require 'bundler/setup'
require 'digest/sha1'
require 'redcarpet'
require 'pygments'
require 'erb'
require 'docverter'
require 'highline'
require 'json'

task :build => [:build_pdf, :build_mobi, :build_epub]
task :check => [:count_hours, :count, :check_tics, :check_todos]
task :default => [:check, :build]

def with_book_dir
  Dir.chdir("/Users/peter/book") do
    yield
  end
end

desc "Count the number of hours spent writing"
task :count_hours do
  hours = 0
  with_book_dir do
    File.open("_hours.md").each do |line|
      if line =~ /\(([\d\.]+)\)/
        hours += $1.to_f
      end
    end
  end

  puts "Total hours: #{hours}"
end

desc "Count words"
task :count do
  with_book_dir do
    goal_count = 30000.0
  
    code_count = 0
    word_count = 0
    
    Dir.glob('*.md').each do |file|
      next if file =~ /^_/
  
      file_code_count = 0
      file_word_count = 0
  
      in_code_block = false
      File.open(file).each do |line|
        if line =~ /^```/
          in_code_block = !in_code_block
          next
        end
  
        if in_code_block
          file_code_count += 1 unless line =~ /^\s+$/
        else
          file_word_count += line.split(/\s+/).compact.size
        end
      end
  
      code_count += file_code_count
      word_count += file_word_count
  
      puts "#{file}: #{file_word_count} #{file_code_count}"
    end
  
    goal_pct = (word_count / goal_count * 100).round
    puts "overall: #{word_count} #{code_count} (#{goal_pct}%)"
  end
end

desc "Check for common annoying things"
task :check_tics do
  with_book_dir do
    count = 0
    Dir.glob('*.md').each do |file|
      next if file =~ /^_/
  
      matches = File.read(file).match(/(Essentially|Basically)/)
      if matches
        count += matches.length
        puts "#{file} matches"
      end
    end
  
    if count > 0
      exit 1
    end
  end
end

desc "Check for any TODO messages"
task :check_todos do
  with_book_dir do
    count = 0
    Dir.glob('*.md').each do |file|
      next if file =~ /^_/
  
      File.open(file).each_with_index do |line, line_num|
        if line =~ /TODO/
          puts "#{file}:#{line_num+1} #{line.rstrip()}"
          count += 1
        end
      end
    end
  
    if count > 0
      exit 1
    end
  end
end

class HTMLwithPygments < Redcarpet::Render::HTML
  def block_code(code, language)
    Pygments.highlight(code, :lexer => language)
  end

  def postprocess(document)
    document.gsub('&#39;', "'")
  end
end

task "Clean the build directory"
task :clean do
  sh("rm -rf build")
end

task "Common build setup"
task :build_common do
  puts "Setting up common stuff"
  Docverter.base_url = 'http://c.docverter.com'

  @raw_contents = ""

  with_book_dir do
    chapters = File.read('chapters').split("\n")
    chapters.each do |chapter|
      @raw_contents << File.read(chapter + ".md")
      @raw_contents << "\n\n"
    end
  end

  @raw_contents_hash = Digest::SHA1.hexdigest(@raw_contents)

  FileUtils.mkdir_p("build")
end

desc "Build a PDF"
task :build_pdf => :build_common do
  puts "Building PDF"
  Docverter.base_url = 'http://c.docverter.com'

  renderer = Redcarpet::Markdown.new(
    HTMLwithPygments.new(:with_toc_data => true),
    :fenced_code_blocks => true,
    :tables => true,
  )

  toc_renderer = Redcarpet::Markdown.new(
    Redcarpet::Render::HTML_TOC
  )

  body = renderer.render(@raw_contents)
  toc = toc_renderer.render(@raw_contents).gsub('&#39;', "'")
  cover = ""

  with_book_dir do
    cover = File.read('cover.md')
  end

  @content = cover + toc + body

  @html = ERB.new(File.read('template.erb')).result(binding)

  html_hash = Digest::SHA1.hexdigest(@html)


  File.open("build/mastering-modern-payments-#{html_hash}.pdf", "w+") do |f|
    pdf = Docverter::Conversion.run do |c|
      c.from    = 'html'
      c.to      = 'pdf'
      c.content = @html

      Dir.glob('assets/*') do |asset|
        c.add_other_file asset
      end
    end
    f.write pdf
  end
end

desc "Build a mobi"
task :build_mobi => :build_common do
  puts "Building MOBI"
  File.open("build/mastering-modern-payments-#{@raw_contents_hash}.mobi", "w+") do |f|
    mobi = Docverter::Conversion.run do |c|
      c.from = 'markdown'
      c.to = 'mobi'
      c.content = @raw_contents
    end

    f.write mobi
  end
end

desc "Build an epub"
task :build_epub => :build_common do
  puts "Building EPUB"
  File.open("build/mastering-modern-payments-#{@raw_contents_hash}.epub", "w+") do |f|
    mobi = Docverter::Conversion.run do |c|
      c.from = 'markdown'
      c.to = 'epub'
      c.content = @raw_contents
    end

    f.write mobi
  end
end

desc "Build everything and upload it"
task :upload => [:check, :clean, :build] do
  hl = HighLine.new
  upload_password = hl.ask("Password: ") { |q| q.echo = false }
  Dir.glob("build/*").each do |file|
    json = `curl -s -k -u admin:#{upload_password} -F files[]=@#{file} https://files.bugsplat.info/upload`
    puts JSON.parse(json)['files'][0]['url']
  end
end
