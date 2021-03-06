require 'date'
require 'erb'
require 'yaml'
include ERB::Util
$no_errors = true
$out_folder_extensions = ['.gif', '.png', '.mp3', '.mp4', '.mov']
$sketch_extensions = ['.gif', '.mp3', '.mp4']
$template_options = {'g' => 'gifEncoder', 'v' => 'audioVideoGenerator'}
$default_description_text = 'Write your description here'
$git_clean_dir = 'working tree clean'
$num_managed_repos = 2

#api
task :status do
	load_config
	print_uncopied_sketches
	print_all_status
end

task :st do
	Rake::Task['status'].invoke
end

task :validate do
	load_config
	validate
end

task :fetch do
	load_config
	if validate
		if !check_new_month?
			fetch_sketches
			generate_files
		end
	else
		puts 'Please fix these errors before fetching'
	end
end

task :clone, [:source, :datestring] do |t, args|
	load_config
	source = dereference args[:source]
	dest = dereference args[:datestring]

	if template_clone_args?(source, dest)
		clone_from_template source, dest

	elsif sketch_clone_args?(source, dest)
		clone_from_sketch source, dest

	else
		puts 'This command clones a new sketch from a template.'
		puts 'Usage:   rake clone[source,destination]'	
		puts 'e.g.     rake clone[gifEncoder,yesterday]'
		puts 'or:      rake clone[audioSequencer,yyyy-mm-dd]'
		puts 'or:      rake clone[v,yesterday]     // \'v\' short for audioVideoGenerator'
		puts 'or:      rake clone[g,yyyy-mm-dd]     // \'g\' short for gifEncoder'
	end
end

task :deploy, :datestring do |t, args|
	load_config
	if args[:datestring] == nil
		puts 'This command deploys one sketch at a time, identified by it\'s date and letter'
		puts 'Usage: rake deploy[2019-12-24a]'
	else
		datestring = args[:datestring].strip.chomp('\n')
		if datestring == ''
			puts 'Usage: rake deploy[2019-12-24a]'
		else
			deploy_all datestring
		end
	end
end

task :jump do
	load_config
	increment_month
end

#config
def load_config
	config = YAML.load_file('config.yml')
	$site_name = config['site_name']
	$sketch_manager_repo = config['sketch_manager_repo']

	$current_month = config['current_month']
	$next_month = get_next_month
	$current_month_name = month_name $current_month
	$next_month_name = month_name $next_month

	$current_sketch_repo = "sketches-#$current_month"
	$sketches_dir = config['sketches_dir']
	$templates_dir = config['templates_dir']
	$jekyll_repo = config['jekyll_repo']
	$site_url = "http://#$jekyll_repo"
	$github_org_name = config['github_org_name']
end

def get_next_month
	actual_current_month = time_from(month_str_from Time.now)
	system_current_month = time_from($current_month)

	next_month = system_current_month >= actual_current_month ?
		system_current_month >> 1 :
		actual_current_month

	month_str_from next_month
end

def time_from month_str
	DateTime.parse "#{month_str}-01"
end

def month_str_from time
	time.strftime '%Y-%m'
end

def month_name month_str
	month_date = time_from month_str
	month_date.strftime '%B'
end

def increment_month
	if new_month_sketch_detected? && ready_for_month_switch?
		puts "Moving to new repo:\n===================\n"
		execute "sed -i '' 's/current_month: #$current_month/current_month: #$next_month/g' config.yml"
		execute "git add config.yml && git commit -m 'Increments to #$next_month' && git push"
		execute "mkdir sketches/sketches-#$next_month"
		execute "cd sketches/sketches-#$next_month && git init && git remote add origin https://github.com/#$github_org_name/sketches-#$next_month.git"
		execute "cp sketches/sketches-#$current_month/.gitignore sketches/sketches-#$next_month/"
		execute "cd sketches/sketches-#$next_month/ && git add .gitignore && git commit -m 'Adds gitignore' && git push -u origin master"
	else
		puts 'You are not ready to jump. Try running \'rake fetch\' for guidance.'
	end
end

#command handling
def execute commandstring
	puts "Executing command: #{commandstring}"
	execute_silent commandstring
end

def execute_silent commandstring
	if $no_errors
		$no_errors = system commandstring
		unless $no_errors
			puts "\nEXECUTION ERROR. Subsequent commands will be ignored\n"
		end
	else
		puts "\nEXECUTION SKIPPED due to previous error\n"
	end
end

#print
def print_all_status
	puts "Sketch repo status:\n===================\n"
	puts "Remote: https://github.com/#$github_org_name/sketches-#$current_month\n"
	puts "Local:  sketches/sketches-#$current_month\n\n"
	execute_silent "cd sketches/#$current_sketch_repo && git status && cd ../.."
	puts "\nJekyll repo status:\n===================\n"
	puts "Remote: https://github.com/#$github_org_name/#$jekyll_repo\n"
	puts "Local:  #$jekyll_repo\n\n"
	execute_silent "cd #$jekyll_repo && git status && cd .."
end

def print_uncopied_sketches
	puts "Source OF sketches:\n===================\n"
	puts 'This checks new sketches only. To deploy edits, just fetch then manually push and deploy them in the appropriate repos.'
	puts
	current_month_sketches = uncopied_sketches $current_month
	next_month_sketches = uncopied_sketches $next_month
	if current_month_sketches.size == 0
		puts "No new #$current_month_name sketches found."
		puts
		if next_month_sketches.size > 0
			puts "NOTE: One or more sketches found for #$next_month_name. Run fetch to begin transition."
			puts
		end
	else
		pluralization = current_month_sketches.size > 1 ? 'es' : ''
		puts "Found #{current_month_sketches.size} new sketch#{pluralization} for #$current_month_name:"
		puts current_month_sketches
		puts
		if !validate
			puts
		end
		if next_month_sketches.size > 0
			puts "NOTE: One or more sketches found for #$next_month_name. Please fetch and deploy all #$current_month_name sketches to jump to #$next_month_name."
			puts
		end
	end
end

#validate (file / directory states)
def validate
	validate_unexpected_assets_not_present && validate_expected_asset_present && validate_snippet_and_description
end

def validate_expected_asset_present
	valid = true
	sketch_dirs.each do |sketch_dir|
		expected_asset_selector = "#{$sketches_dir}/#{sketch_dir}/bin/data/out/#{sketch_dir}.*"
		if Dir.glob(expected_asset_selector).empty?
			puts "WARNING: No valid asset found in 'bin/data/out' of sketch #{sketch_dir}"
			valid = false
		end
	end
	valid
end

def validate_unexpected_assets_not_present
	valid = true
	sketch_dirs.each do |sketch_dir|
		all_asset_selector = "#{$sketches_dir}/#{sketch_dir}/bin/data/out/*.*"
		Dir.glob(all_asset_selector).select do |entry|
			if File.basename(entry, '.*') != sketch_dir || !$out_folder_extensions.include?(File.extname(entry))
				puts "WARNING: Unexpected asset '#{File.basename entry}' found in 'bin/data/out' of sketch #{sketch_dir}"
				valid = false
			end
		end
	end
	valid
end

def validate_snippet_and_description
	valid = true
	sketch_dirs.each do |sketch_dir|
		already_copied_file = Dir.glob "sketches/*/#{sketch_dir}/src/ofApp.cpp"
		if already_copied_file.empty?
			contents = read_snippet_contents sketch_dir
			if contents.nil? || contents.empty? || contents == $default_description_text
				puts "WARNING: Snippet not found for sketch #{sketch_dir}"
				valid = false
			end
			contents = read_description_contents sketch_dir
			if contents.nil? || contents.empty? || contents == $default_description_text
				puts "WARNING: Description not found for sketch #{sketch_dir}"
				valid = false
			end
		end
	end
	valid
end

def sketch_dirs
	Dir.entries($sketches_dir).select do |entry|
		File.directory? File.join($sketches_dir, entry) and !(entry == '.' || entry == '..')
	end
end

#fetch
def check_new_month?
	if new_month_sketch_detected?
		if ready_for_month_switch?
			puts "Congratulations! It's time to switch from your #$current_month_name repo to a new #$next_month_name repo. We must do this once a month, to keep your repos a reasonable size."
			puts
			puts "NOTE: Your new #$next_month_name sketch has not been fetched because it must be fetched to a new #$next_month_name repo, once the repo is created."
			puts
			puts "Just follow the simple steps below, and then you will be able to continue working as normal, but now deploying to a new #$next_month_name repo:"
			puts
			puts "1. Go to https://github.com/organizations/#$github_org_name/repositories/new"
			puts "2. Enter sketches-#$next_month as the repository name and click 'create repository'"
			puts "3. Run 'rake jump' to finalize the switch to #$next_month_name"
			puts "4. Run 'rake fetch' to fetch your new #$next_month_name sketch and continue working as normal"
			true
		else
			puts "WARNING: One or more sketches for #$next_month_name were detected, but will not be fetched yet. To fetch them, please deploy all #$current_month_name sketches and run 'rake fetch' again."
			false
		end
	else
		false
	end
end

def new_month_sketch_detected?
	uncopied_sketches($next_month).length > 0
end

def ready_for_month_switch?
	uncopied_sketches($current_month).length == 0 && !undeployed_sketches_exist?
end

def uncopied_sketches target_month
	found_uncopied = Array.new
	Dir.glob "#$sketches_dir/#{target_month}*" do |dirpath|
		sketch_dir = File.basename dirpath
		target_filepath = "sketches/sketches-#{target_month}/#{sketch_dir}/src/ofApp.cpp"
		unless File.exist?(target_filepath)
			found_uncopied.push sketch_dir
		end
	end
	found_uncopied
end

def undeployed_sketches_exist?
	system_output = `rake status`
	return system_output.scan($git_clean_dir).length != $num_managed_repos
end

def fetch_sketches
	starttime = Time.now
	print "Fetching openFrameworks sketches... "
	execute_silent "rsync -ru #$sketches_dir/#$current_month* sketches/#$current_sketch_repo"
	endtime = Time.now
	print "completed in #{endtime - starttime} seconds.\n"
end

#clone
def dereference arg
	result = arg || ''
	result = result.size == 1 ? $template_options[result] : result
	result = result == 'today' ? Date::today.strftime : result
	result = result == 'yesterday' ? Date::today.prev_day.strftime : result
	result
end

def template_clone_args? source, dest
	valid_template?(source) && valid_date?(dest)
end

def sketch_clone_args? source, dest
	(valid_date?(source) || valid_date?(source.chop)) && valid_date?(dest) && source != dest
end

def clone_from_template source, dest
	source = "example-#{source}"
	source_path = "#$templates_dir/#{source}"
	clone source, source_path, 'template', dest
end

def clone_from_sketch source, dest
	source_path = "#$sketches_dir/#{source}"
	clone source, source_path, 'sketch', dest
end

def clone source, source_path, source_type, dest
	dest = ensure_valid_dest dest
	dest_path = "#$sketches_dir/#{dest}"

	if valid_clone_source?(source, source_path)
		#copy the template
		execute_silent "rsync -ruq #{source_path}/ #{dest_path}"

		#rename files
		xcodeproj = "#{dest_path}/#{source}.xcodeproj"
		execute_silent "mv #{xcodeproj}/xcshareddata/xcschemes/#{source}\\\ Debug.xcscheme   #{xcodeproj}/xcshareddata/xcschemes/#{dest}\\\ Debug.xcscheme || true"
		execute_silent "mv #{xcodeproj}/xcshareddata/xcschemes/#{source}\\\ Release.xcscheme #{xcodeproj}/xcshareddata/xcschemes/#{dest}\\\ Release.xcscheme || true"
		execute_silent "mv #{xcodeproj} #{dest_path}/#{dest}.xcodeproj"
		xcodeproj = "#{dest_path}/#{dest}.xcodeproj"

		#recursive rewrite of references to old filenames
		execute_silent "cd #{xcodeproj} && LC_ALL=C find . -path '*.*' -type f -exec sed -i '' -e 's/#{source}/#{dest}/g' {} +"

		#clear project instance-specific & user data
		execute_silent "rm -f  #{xcodeproj}/project.xcworkspace/xcshareddata/*.xccheckout"
		execute_silent "rm -rf #{xcodeproj}/project.xcworkspace/xcuserdata"
		execute_silent "rm -rf #{xcodeproj}/xcuserdata"
		execute_silent "rm -rf #$sketches_dir/#{dest}/bin/data/AudioUnitPresets/.trash/"

		#clear generated binaries (note that .app files can be packages)
		execute_silent "rm -rf #{dest_path}/bin/*.app"
		$out_folder_extensions.each do |ext|
			execute_silent "rm -f #{dest_path}/bin/data/out/*#{ext}"
		end

		#remove comments, load snippet headings
		cpp_path = "#{dest_path}/src/ofApp.cpp"
		contents = File.read(cpp_path)
		file = File.new(cpp_path, "w+")
		File.open(file, "a") do |file|
			contents = contents.gsub(/[\n\r]*.*\/\/.*/, '')
			contents = contents.gsub(/[\n\r]\/\*.*?\*\//m, '')
			contents = contents.gsub("\"out/filename\"", "\"out/#{dest}\"")
			contents = contents.gsub("\nvoid ofApp::setup(){", "/* Begin description\n{\n    #$default_description_text\n}\nEnd description */\n\n/* Snippet begin */\nvoid ofApp::setup(){")
			contents = contents.gsub("}\n\nvoid ofApp::keyPressed(int key){", "}\n/* Snippet end */\n\nvoid ofApp::keyPressed(int key){")
			file.write contents
		end

		puts "Created sketch #{dest} by cloning #{source_type} #{source}."
	end
end

def valid_clone_source? source, source_path
	result = true
	unless Dir.exist?(source_path)
		puts "WARNING: Source for #{source} not found... aborting."
		result = false
	end
	result
end

def ensure_valid_dest dest
	dest_path = "#$sketches_dir/#{dest}"
	letter = 'a'
	while Dir.exist?("#{dest_path}#{letter}")
		letter.next!
	end
	"#{dest}#{letter}"
end

def valid_date? arg
	begin
	   Date.parse(arg) && /^\d{4}-\d{2}-\d{2}$/.match(arg)
	rescue ArgumentError
	   false
	end
end

def valid_template? arg
	$template_options.has_value?(arg)
end

#deploy
def deploy_all datestring
	puts "\nDeploying sketch:\n================="
	execute "cd sketches/#$current_sketch_repo && pwd && git add #{datestring} && git commit -m 'Adds sketch #{datestring}' && git push & cd ../.."
	puts "\nDeploying jekyll:\n================="
	execute "cd #$jekyll_repo && pwd && git add app/_posts/#{jekyll_sketch_url datestring}.md && git commit -m 'Adds sketch #{datestring}' && git push && grunt deploy"
end

#generate
def generate_files
	starttime = Time.now
	print "Generating jekyll post files... "
	Dir.glob "sketches/#$current_sketch_repo/*/bin/data/out/*.*" do |filename|
		if filename.end_with? '.mov'
			filename_converted = filename.gsub(/.mov$/, '.mp4')
			execute_silent "ffmpeg -i #{filename} #{filename_converted} && rm #{filename}"
			filename = filename_converted
		end
		$sketch_extensions.each do |ext|
			if filename.end_with? ext
				filename = File.basename(filename, ext)
				ext.delete! '.'
				generate_post filename, ext
				generate_readme filename, ext
			end
		end
	end
	endtime = Time.now
	print "completed in #{endtime - starttime} seconds.\n"
end

def generate_post datestring, ext
	filepath = "#$jekyll_repo/app/_posts/#{jekyll_sketch_url datestring}.md"
	unless File.exist?(filepath)
		file = open(filepath, 'w')
		file.write(post_file_contents datestring, ext)
		file.close
	end
end

def generate_readme datestring, ext
	filepath = "sketches/#$current_sketch_repo/#{datestring}/README.md"
	unless File.exist?(filepath)
		file = open(filepath, 'w')
		file.write(readme_file_contents datestring, ext)
		file.close
	end
end

#read from disk
def read_snippet_contents sketch_dir
	regex = /\/\* Snippet begin \*\/(.*?)\/\* Snippet end \*\//m
	read_cpp_contents sketch_dir, regex
end

def read_description_contents sketch_dir
	regex = /\/\* Begin description\n\{(.*?)\}\nEnd description \*\//m
	read_cpp_contents sketch_dir, regex
end

def read_cpp_contents sketch_dir, regex
	source_cpp_path = "#{$sketches_dir}/#{sketch_dir}/src/ofApp.cpp"
	if File.exist?(source_cpp_path)
		file = open(source_cpp_path, 'r')
		contents = file.read
		file.close
		selected = contents[regex, 1]
		if selected.nil?
			selected
		else
			html_escape(selected.strip.chomp('\n'))
		end
	else
		nil
	end
end

#string builders
def post_file_contents datestring, ext
	<<-eos
---
layout: post
title:  "Sketch #{datestring}"
date:   #{datestring}
---
<div class="code">
    <ul>
		<li><a href="{% post_url #{jekyll_sketch_url datestring} %}">permalink</a></li>
		<li><a href="https://github.com/#$github_org_name/#$current_sketch_repo/tree/master/#{datestring}">code</a></li>
		<li><a href="#" class="snippet-button">show snippet</a></li>
	</ul>
    <pre class="snippet">
        <code class="cpp">#{read_snippet_contents(datestring)}</code>
    </pre>
</div>
<p class="description">#{read_description_contents(datestring)}</p>
#{ext == 'gif' ? render_post_gif(datestring) : ''}
#{ext == 'mp3' ? render_post_mp3(datestring) : ''}
#{ext == 'mp4' ? render_post_mp4(datestring) : ''}
eos
end

def readme_file_contents datestring, ext
	<<-eos
Sketch #{datestring}
--
This subfolder of the [#$current_sketch_repo repo](https://github.com/#$github_org_name/#$current_sketch_repo) is the root of an individual openFrameworks sketch. It contains the full source code used to generate this sketch:

#{ext == 'gif' ? render_readme_gif(datestring) : ''}
#{ext == 'mp3' ? render_readme_mp3(datestring) : ''}
#{ext == 'mp4' ? render_readme_mp4(datestring) : ''}

This source code is published automatically along with each sketch I add to [#$site_name](#$site_url). Here is a [permalink to this sketch](#$site_url/sketch-#{reverse datestring}/) on the #$site_name site.

Run this yourself
--
If you are running [openFrameworks via XCode](http://openframeworks.cc/download/) on a Mac you can just clone this directory and launch the `xcodeproj`. If you do that you should see something similar to the sketch above.

Addons
--
If the sketch uses any [addons](http://www.ofxaddons.com/unsorted) you don't already have you will have to clone them. Note that this readme was auto-generated and so doesn't list the addon dependencies. However you can figure out which addons you need pretty easily.

In XCode you will see a panel like this. Expand the folders under `addons` until you can see some of the source files underneath.

![How to find missing addon dependencies](https://github.com/#$github_org_name/#$sketch_manager_repo/blob/master/images/dependencies.png?raw=true)

In the example above, the addon `ofxLayerMask` is missing (it's source files are red), but `ofxGifEncoder` is present.

Versions
--
Note that openFrameworks doesn't have a great system for versioning addons. If you are getting results that look different to the gif above, or it won't compile, you may have cloned newer versions of some addons than were originally used to generate the sketch.

In that case you should clone the addon(s) at the most recent commit before the sketch date. Not ideal, but you will only have to do it rarely.

Yes, openFrameworks could use a good equivalent of [bundler](http://bundler.io/). You should write one!
eos
end

#string builder helpers
def jekyll_sketch_url datestring
	letter = datestring[-1]
	dateonly = datestring.chop
	return "#{dateonly}-sketch-#{letter}"
end

def reverse datestring
	date = DateTime.parse(datestring)
	date.strftime('%d-%m-%Y')
end

def raw_url datestring, ext
	"https://github.com/#$github_org_name/#$current_sketch_repo/blob/master/#{datestring}/bin/data/out/#{datestring}.#{ext}?raw=true"
end

def render_post_gif datestring
	"![Daily sketch](#{raw_url datestring, 'gif'})"
end

def render_post_mp3 datestring
	<<-eos
<p>
	<img src="#{raw_url datestring, 'png'}" alt="Sketch #{datestring}">
	<audio controls>
		<source src="#{raw_url datestring, 'mp3'}" type="audio/mpeg">
		Your browser does not support the audio element.
	</audio>
</p>
eos
end

def render_post_mp4 datestring
	<<-eos
<p>
	<video controls>
		<source src="#{raw_url datestring, 'mp4'}" type="video/mp4">
		Your browser does not support the video element.
    </video>
</p>
eos
end

def render_readme_gif datestring
	"![Sketch #{datestring}](#{raw_url datestring, 'gif'})"
end

def render_readme_mp3 datestring
	<<-eos
![Sketch #{datestring}](#{raw_url datestring, 'png'})
[Listen to the sketch on #$site_name](#$site_url/sketch-#{reverse datestring}/)"
eos
end

def render_readme_mp4 datestring
	<<-eos
[View the sketch on #$site_name](#$site_url/sketch-#{reverse datestring}/)"
eos
end