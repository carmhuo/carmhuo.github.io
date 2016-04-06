task :default => :new

require 'fileutils'

desc "create new post"
task :new do
  puts "Please input POST url:"
	@url = STDIN.gets.chomp
	puts "Please input title of POST:"
	@name = STDIN.gets.chomp
	puts "Please input subtitle of POST:"
	@subtitle = STDIN.gets.chomp
	puts "Please input categories:"
	@categories = STDIN.gets.chomp
	puts "Please input tag of POST:"
	@tag = STDIN.gets.chomp
	@slug = "#{@url}"
	@slug = @slug.downcase.strip.gsub(' ', '-')
	@date = Time.now.strftime("%F")
	@post_name = "_posts/#{@date}-#{@slug}.md"
	if File.exist?(@post_name)
			abort("filename does existed,Create failure!")
	end
	FileUtils.touch(@post_name)
	open(@post_name, 'a') do |file|
			file.puts "---"
			file.puts "layout: post"
			file.puts "title: #{@name}"
			file.puts "subtitle: #{@subtitle}"
			file.puts "author: carm"
			file.puts "date: #{Time.now}"
			file.puts "header-img: img/home-pg-city.jpg"
			file.puts "categories: #{@categories}"
			file.puts "tag: #{@tag}"
			file.puts "---"
	end
	exec "vi #{@post_name}"
end
