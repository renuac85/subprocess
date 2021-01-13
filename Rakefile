require 'bundler/gem_tasks'
require 'bundler/setup'
require 'rake/testtask'

require 'tempfile'

task :default do
  sh 'rake -T'
end

Rake::TestTask.new do |t|
  t.libs.push "lib"
  t.test_files = FileList['test/test_*.rb']
  t.verbose = true
end

task :sord do
  rbi_file = 'rbi/subprocess.rbi'
  sh 'sord', '--no-sord-comments', rbi_file

  puts 'Running substitutions on sord output'
  Tempfile.create do |tmp|
    File.open(rbi_file) do |f|
      f.each.with_index do |line, idx|
        # Sord includes trailing whitespace on certain lines.
        line.rstrip!

        # Stripe's codebase treats top-level module definitions specially (for
        # the sake of modularity and packaging). This makes it easier to work
        # with Stripe's packager.
        line.sub!(/^module Subprocess$/, "module ::Subprocess")

        # Currently there is no way to specify "block, but optional" in YARD
        # https://github.com/AaronC81/sord/issues/129
        # ... but all of our blocks are optional, as of time of writing.
        if line =~ /blk:/
          proc_type = 'T.proc.params(process: Process).void'
          line.sub!(proc_type, "T.nilable(#{proc_type})")
        end

        # Technically speaking, the YARD docs are right for the return type of
        # `communicate`. It returns `nil` if given a block, otherwise a tuple.
        #
        # But there were too many useless errors caused by this in Stripe's
        # codebase, and Sorbet doesn't support overloads, so for sake of
        # convenience we treat this return as never returning `nil`.
        #
        # Currently, `communicate` is the only method that returns a tuple of
        # Strings.
        line.sub!('T.nilable([String, String])', '[String, String]')

        tmp.puts(line)

        if idx == 0
          tmp.puts("")
          tmp.puts("# THIS FILE IS AUTOGENERATED. DO NOT EDIT.")
          tmp.puts("# To regenerate from YARD comments:")
          tmp.puts("#")
          tmp.puts("#     bundle exec rake sord")
          tmp.puts("#")
        end
      end
    end
    File.rename(tmp.path, rbi_file)
  end
  puts 'Done running substitutions on sord output'
end

task :publish do
  require_relative 'lib/subprocess'
  sh 'gem', 'build', 'subprocess.gemspec'
  gem_file = "subprocess-#{Subprocess::VERSION}.gem"
  Bundler.with_unbundled_env do
    api_key = Subprocess.check_output(['fetch-password', '--quiet', '--raw', 'bindings/rubygems-api-key']).strip
    gem_env = ENV.to_hash.update('GEM_HOST_API_KEY' => api_key)
    Subprocess.check_output(['gem', 'push', gem_file], env: gem_env)
  end
  sh 'rm', '-f', gem_file
end
