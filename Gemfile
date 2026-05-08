source "https://rubygems.org"

# Jekyll itself
gem "jekyll", "~> 4.3"

# GitHub Pages-compatible plugins
group :jekyll_plugins do
  gem "jekyll-remote-theme"
  gem "jekyll-seo-tag"
  gem "jekyll-sitemap"
end

# Windows / JRuby support
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", "~> 2.0"
  gem "tzinfo-data"
end

# Performance fix for some platforms
gem "wdm", "~> 0.1.1", :platforms => [:mingw, :x64_mingw, :mswin]

# CSV support for newer Ruby (just-the-docs uses Lunr; this avoids a warning)
gem "csv"
gem "base64"
gem "logger"
