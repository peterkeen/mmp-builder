require 'bundler/setup'
require 'digest/sha1'
require 'redcarpet'
require 'redcarpet/render_strip'
require 'pygments'
require 'erb'
require 'docverter'
require 'highline'
require 'json'
require 'httparty'
require 'zip/zipfilesystem'

desc "Build everything"
task :build => ['build:html', 'build:pdf', 'build:mobi', 'build:epub', 'build:app']
task 'build:full' => ['check', 'build:clean', :build]
task 'package' => ['package:basic', 'package:deluxe']

desc "Basic sanity check"
task :check => ['check:count_hours', 'check:count', 'check:tics', 'check:todos', 'check:syntax']

desc "Sanity checks plus syntax check"

task :default => :check

def with_book_dir
  Dir.chdir("/Users/peter/book") do
    yield
  end
end

class RubySyntaxChecker < Redcarpet::Render::Base

  attr_reader :failed

  def check_ruby(ruby_code, raw)
    eval("BEGIN {return true}\n#{ruby_code}", nil, '<check>', 0)
  rescue SyntaxError
    puts raw
    puts $!.message
    @failed = true
  end

  def block_code(code, language)
    if language == 'ruby'
      check_ruby(code, code)
      ""
    end
  end
end

class LinkChecker < Redcarpet::Render::Base
  def link(href, title, content)
    begin
      return "" unless href =~ /^http/
      response = HTTParty.head(href)
      if response.code != 200
        puts "Error: #{href}: #{response.code}"
      end
    rescue Errno::ECONNREFUSED
      puts "Connection refused: #{href} #{title} #{content}"
    end
  end
end

class HTMLWithChapterNumberingAndPygments < Redcarpet::Render::HTML
  def header(text, header_level)
    if header_level == 1
      @counter ||= 0
      @counter += 1
      "<h1 id=\"chapter_#{@counter}\"><small>Chapter #{@counter}</small><br>#{text}</h1>\n"
    else
      "<h#{header_level}>#{text}</h#{header_level}>\n"
    end
  end
  def block_code(code, language)
    Pygments.highlight(code, :lexer => language)
  end

  def postprocess(document)
    document.gsub('&#39;', "'")
  end
end

class TOCWithChapterNumbering < Redcarpet::Render::StripDown
  def header(text, header_level)
    return unless header_level == 1
    @chapters ||= []
    @chapters << text
    ""
  end

  def postprocess(document)
    items = []
    @chapters.each_with_index do |text, i|
      items << "  <li><a href=\"#chapter_#{(i+1).to_s}\">#{text}</a></li>"
    end
    return <<HERE
<ol>
#{items.join("\n")}
</ol>
HERE
  end
end

namespace :check do
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
      block_count = 0
      word_count = 0

      Dir.glob('*.md').each do |file|
        next if file =~ /^_/

        file_code_count = 0
        file_word_count = 0
        file_block_count = 0

        in_code_block = false
        File.open(file).each do |line|
          if line =~ /^```/
            in_code_block = !in_code_block
            if in_code_block
              file_block_count += 1
            end
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
        block_count += file_block_count

        puts "#{file}: #{file_word_count} #{file_code_count} #{file_block_count}"
      end

      goal_pct = (word_count / goal_count * 100).round
      puts "overall: #{word_count} #{code_count} #{block_count} (#{goal_pct}%)"
    end
  end

  desc "Check for common annoying things"
  task :tics do
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

  desc "Check syntax of ruby blocks"
  task :syntax do
    with_book_dir do
      Dir.glob("*.md").each do |file|
        next if file =~ /^_/
        checker = RubySyntaxChecker.new
        renderer = Redcarpet::Markdown.new(checker, :fenced_code_blocks => true)
        renderer.render(File.read(file))
        if checker.failed
          puts file
        end
      end
    end
  end

  desc "Check links"
  task :links do
    with_book_dir do
      Dir.glob("*.md").each do |file|
        next if file =~ /^_/
        renderer = Redcarpet::Markdown.new(LinkChecker.new, :fenced_code_blocks => true)
        renderer.render(File.read(file))
      end
    end
  end


  desc "Check for any TODO messages"
  task :todos do
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
end

namespace :build do
  desc "Clean the build directory"
  task :clean do
    sh("rm -rf build")
  end

  task "Common build setup"
  task :common do
    puts "Setting up common stuff"
    Docverter.base_url = 'http://c.docverter.com'

    @raw_contents = ""

    with_book_dir do
      chapters = File.read('chapters').split("\n")
      chapters.each do |chapter|
        next if ENV['chapter'] && chapter != ENV['chapter']
        @raw_contents << File.read(chapter + ".md")
        @raw_contents << "\n\n"
      end
    end

    @raw_contents_hash = Digest::SHA1.hexdigest(@raw_contents)
    @build_dir = "build/#{@raw_contents_hash}"
    FileUtils.mkdir_p(@build_dir)
  end

  task :html => 'build:common' do
    puts "Building HTML"
    renderer = Redcarpet::Markdown.new(
      HTMLWithChapterNumberingAndPygments.new,
      :fenced_code_blocks => true,
      :tables => true,
    )

    toc_renderer = Redcarpet::Markdown.new(
      TOCWithChapterNumbering
    )

    body = renderer.render(@raw_contents)
    toc = toc_renderer.render(@raw_contents)
    cover = ""

    with_book_dir do
      cover = File.read('_cover.md')
    end

    @content = cover + toc + body
    @html = ERB.new(File.read('site_template.erb')).result(binding)
    FileUtils.mkdir_p(File.join(@build_dir, 'html'))
    sh "cp assets/* #{@build_dir}/html"
    File.open("#{@build_dir}/html/Mastering Modern Payments.html", "w+") do |f|
      f.write @html
    end

  end

  desc "Build a PDF"
  task :pdf => 'build:common' do
    puts "Building PDF"
    Docverter.base_url = 'http://c.docverter.com'

    renderer = Redcarpet::Markdown.new(
      HTMLWithChapterNumberingAndPygments.new(:with_toc_data => true),
      :fenced_code_blocks => true,
      :tables => true,
    )

    toc_renderer = Redcarpet::Markdown.new(
      TOCWithChapterNumbering
    )

    body = renderer.render(@raw_contents)
    toc = toc_renderer.render(@raw_contents).gsub('&#39;', "'")
    cover = ""
    cover_sample = ""

    with_book_dir do
      cover = File.read('_cover.md')
      cover_sample = File.read('_cover_sample.md')
    end

    if ENV['chapter']
      @content = cover_sample + body
    else
      @content = cover + "<h1>Table of Contents</h1>" + toc + body
    end

    @html = ERB.new(File.read('template.erb')).result(binding)

    html_hash = Digest::SHA1.hexdigest(@html)


    File.open("#{@build_dir}/Mastering Modern Payments.pdf", "w+") do |f|
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
  task :mobi => 'build:common' do
    puts "Building MOBI"
    File.open("#{@build_dir}/Mastering Modern Payments.mobi", "w+") do |f|
      mobi = Docverter::Conversion.run do |c|
        c.from = 'markdown'
        c.to = 'mobi'
        c.content = @raw_contents
      end

      f.write mobi
    end
  end

  desc "Build an epub"
  task :epub => 'build:common' do
    puts "Building EPUB"
    File.open("#{@build_dir}/Mastering Modern Payments.epub", "w+") do |f|
      mobi = Docverter::Conversion.run do |c|
        c.from = 'markdown'
        c.to = 'epub'
        c.content = @raw_contents
      end

      f.write mobi
    end
  end

  desc "Export a zip file of the sales app"
  task :app => 'build:common' do
    puts "Exporting source"
    Dir.chdir(@build_dir) do
      sh 'git archive --remote git@git.bugsplat.info:peter/sales.git --format zip master -o sales.zip'
    end
  end

end


def make_zip_file(name, files)
  Zip::ZipFile.open(name, Zip::ZipFile::CREATE) do |zf|
    files.each do |file|
      puts file
      zf.file.open(file, "w") { |f| f.write File.read(file) }
    end
  end
end

namespace :package do
  task :basic => 'build:full' do
    Dir.chdir(@build_dir) do
      make_zip_file("mastering_modern_payments.zip", [
          'Mastering Modern Payments.pdf',
          'Mastering Modern Payments.mobi',
          'Mastering Modern Payments.epub',
          'Mastering Modern Payments.html',
      ])
    end
  end

  task :deluxe => 'build:full' do
    Dir.chdir(@build_dir) do
      make_zip_file("mastering_modern_payments_deluxe.zip", [
          'Mastering Modern Payments.pdf',
          'Mastering Modern Payments.mobi',
          'Mastering Modern Payments.epub',
          'Mastering Modern Payments.html',
          'sales.zip',
      ])
    end
  end
end
