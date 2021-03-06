require 'digest'
require 'erb'
require 'net/http'
require 'tmpdir'

def fetch(http, req, limit = 10)
  raise 'too many HTTP redirects' if limit == 0
  resp = http.get(req)

  case resp
  when Net::HTTPSuccess then resp
  when Net::HTTPRedirection then fetch(http, resp['location'], limit - 1)
  else raise "Request for listing failed: #{resp.body}"
  end
end

PKG_TO_COLLECTIONS = {
  'puppet-bolt' => 'puppet',
  'pdk' => 'puppet'
}

def operating_systems(collection)
  collection == 'puppet5' ? %w[10.10 10.11 10.12 10.13 10.14] : %w[10.11 10.12 10.13 10.14]
end

namespace :brew do
  desc 'Update SHAs for a specific package: rake brew:cask[puppet-bolt] or rake brew:cask[puppet-agent,5]'
  task :cask, [:pkg, :collection] do |task, args|
    pkg = args[:pkg]
    collection = PKG_TO_COLLECTIONS[pkg] || "puppet#{args[:collection]}"
    cask = pkg
    cask += '-' + args[:collection] if args[:collection]
    os_versions = operating_systems(collection)

    host = 'downloads.puppet.com'
    path_pre = "/mac/#{collection}/"
    package_triples = Net::HTTP.start(host, use_ssl: true) do |http|
      latest_versions = os_versions.map do |os_ver|
        resp = fetch(http, "#{path_pre}#{os_ver}/x86_64")
        raise "Request for listing failed: #{resp.body}" unless resp.kind_of? Net::HTTPSuccess
        versions = resp.body.scan(/#{pkg}-(\d+\.\d+\.\d+(?:\.\d+)?)-\d\.osx#{os_ver}\.dmg/).map(&:first).uniq
        versions.sort_by! {|version| Gem::Version.new(version)}
        versions.last
      end
      puts "Getting SHA256 sums for #{pkg}: #{latest_versions}"

      os_versions.zip(latest_versions).map do |os_ver, pkg_ver|
        if pkg_ver
          resp = http.get("#{path_pre}#{os_ver}/x86_64/#{pkg}-#{pkg_ver}-1.osx#{os_ver}.dmg")
          sha = Digest::SHA256.hexdigest(resp.body)
          [os_ver, pkg_ver, sha]
        end
      end.compact
    end

    url = "https://#{host}#{path_pre}"+'#{os_ver}/x86_64/'+pkg+'-#{version}-1.osx#{os_ver}.dmg'

    source_stanza = ERB.new(File.read(File.join(__dir__, 'templates', "source_stanza.erb")), 0, '-').result(binding)

    cask_erb = ERB.new(File.read(File.join(__dir__, 'templates', "#{cask}.rb.erb")), 0, '-')
    File.write(File.join(__dir__, 'Casks', "#{cask}.rb"), cask_erb.result(binding))
  end
end

