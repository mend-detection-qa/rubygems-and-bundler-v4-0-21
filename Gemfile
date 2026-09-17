# frozen_string_literal: true

source "https://rubygems.org"

# Registry gem — checksum validation probe target.
# Bundler 4.0.21 changed handling of empty CHECKSUMS entries; this gem
# exercises the new behaviour via its lock-file checksum slot.
gem "rack", "~> 3.1"

# Registry gem with a transitive dependency, to widen the tree.
gem "sinatra", "~> 4.0"

# Git-sourced gem — exercises Bundler 4.0.21's
# safe.bareRepository=explicit security configuration applied when
# cloning / fetching git sources during resolution.
gem "rack-test", git: "https://github.com/rack/rack-test.git",
                 tag: "v2.1.0"
