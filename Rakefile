PLATFORMS = [:linux, :osx, :win]
JVSTWRAPPER_VERSION = '0.9g'

def bundle_url(platform)
  "http://freefr.dl.sourceforge.net/sourceforge/jvstwrapper/jVSTwRapper-Release-#{JVSTWRAPPER_VERSION}-#{platform}.zip"
end

def system!(cmd)
	puts "Launching #{cmd}"
  raise "Failed to launch #{cmd}" unless system(cmd)
end

def templatized_file(source,target)
  File.open(target,"w") do |output|
    IO.readlines(source).each do |line|
      line = yield line
      output << line
    end
  end
end

def template(platform)
  "templates/#{platform}"
end

def download_and_unpack(platform, unzip_folder)
  url = bundle_url(platform)
  zip_file = unzip_folder + "/" + url.split('/').last
  system!("curl #{url} -o #{zip_file} --silent --show-error --location")
  system!("unzip -q #{zip_file} -d #{unzip_folder}")
end

namespace :prepare do
  
  desc "Download and prepare the platform-specific templates"
  task :templates do
    rm_rf "templates"
    PLATFORMS.each do |platform|
      unzip_folder = template(platform) + "/archive/unzipped"
      mkdir_p unzip_folder unless File.exist?(unzip_folder)
      download_and_unpack(platform, unzip_folder)
    
      root = "#{unzip_folder}/jVSTwRapper-Release-#{JVSTWRAPPER_VERSION}"
      if platform == :osx
        cp_r root+"-osx/jvstwrapper.vst", template(platform)
        File.rename(template(platform) + "/jvstwrapper.vst", template(platform) + '/wrapper.vst')
      else
        mkdir template(platform) + "/wrapper.vst"
        Dir[root+"/*.*"].grep(/(dll|so)$/).each { |f| cp f, template(platform) + "/wrapper.vst" }
      end
      rm_rf template(platform) + "/archive"
      Dir[template(platform) + "/**/*.*"].each do |f|
        rm f if f =~ /\.(bmp|ini|jar)$/
        mv f, File.dirname(f) + "/wrapper.#{$1}" if f =~ /\.(so|jnilib|dll)$/
      end
    end
  end

  desc "Download required libs"
  task :libs do
    rm_rf "libs"
    mkdir_p "libs/temp"
    # the jars are shared accross all distributions, so pick one and extract them
    download_and_unpack(:win, "libs/temp")
    Dir["libs/temp/**/*-#{JVSTWRAPPER_VERSION}.jar"].select { |e| e.split('/').last =~ /jvst(system|wrapper)/i }.each { |f| cp f, "libs" }

    #{}"http://repository.codehaus.org/org/jruby/jruby/1.2.0/"
    system!("curl http://repository.codehaus.org/org/jruby/jruby-complete/1.2.0/jruby-complete-1.2.0.jar -o libs/jruby-complete-1.2.0.jar --silent --show-error")
  end
  
end

task :environment do
  @plugin_name = ENV['plugin']
  abort("Specify a plugin with 'rake compile package deploy plugin=Delay'") unless @plugin_name
  @plugin_folder = "plugins/#{@plugin_name}"
  @jars = Dir["libs/*.jar"]
end

desc "Clean previous build (.class, /build)"
task :clean => :environment do
  Dir[@plugin_folder + "/*.class"].each { |f| rm f }
  rm_rf @plugin_folder + "/build"
end

desc "Compile the plugin"
task :compile => [:environment,:clean] do
	system!("javac src/*.java -classpath #{@jars.join(':')}")
end

desc "Package the plugin for each platform"
task :package => :environment do
  rm_rf @plugin_folder + "/build"
  mkdir @plugin_folder + "/build"
  PLATFORMS.each do |platform|
    build_folder = @plugin_folder + "/build/#{platform}"
    resources_folder = build_folder + "/wrapper.vst" + (platform == :osx ? "/Contents/Resources" : "")
    
    # copy platform template
    cp_r template(platform), build_folder

    # create ini file
    ini_file = resources_folder + "/" + (platform == :osx ? "wrapper.jnilib.ini" : "wrapper.ini")
    File.open(ini_file,"w") do |output|
      content = [ "PluginClass=JRubyVSTPluginProxy",
                  #"PluginUIClass=jvst/examples/jaydlay/JayDLayGUI"
                  "ClassPath=" + @jars.reject { |f| f =~ /jVSTsYstem/}.map { |e| "{WrapperPath}/"+ e.split('/').last }.join(':'),
                  "SystemClassPath={WrapperPath}/jVSTsYstem-0.9g.jar",
                  "IsLoggingEnabled=1",
                  "RubyPlugin=#{@plugin_name}"]
      content.each { |e| output << e + "\n"}
    end
    
    # add classes and jars
    (@jars + Dir['src/*.class'] + Dir[@plugin_folder + "/*.rb"]).each { |f| cp f, resources_folder }

    # create Info.plist (osx only)
    if platform == :osx
      plist_file = build_folder + "/wrapper.vst/Contents/Info.plist"
      plist_content = IO.read(plist_file).gsub!(/<key>(\w*)<\/key>\s+<string>([^<]+)<\/string>/) do
        key,value = $1, $2
        value = @plugin_name+".jnilib" if key == 'CFBundleExecutable'
        value = @plugin_name if key == 'CFBundleName'
        "<key>#{key}</key>\n	<string>#{value}</string>"
      end
      File.open(plist_file,"w") { |output| output << plist_content }
    end

    # rename to match plugin name - two pass - first the directories, then the files
    (0..1).each do |pass|
      Dir[build_folder+"/**/wrapper*"].partition { |f| File.directory?(f) }[pass].each do |file|
        File.rename(file,file.gsub(/wrapper/,@plugin_name))
      end
    end
  end
end

desc "Deploy the plugin"
task :deploy => [:environment] do#, :package] do
  #target_folder = "/Library/Audio/Plug-Ins/VST/"
  target_folder = File.expand_path("~/VST-Dev")
  Dir["#{@plugin_folder}/build/osx/*"].each do |plugin|
    target_plugin = "#{target_folder}/#{plugin.split('/').last}"
    rm_rf(target_plugin) if File.exist?(target_plugin)
    cp_r plugin, target_plugin
  end

end