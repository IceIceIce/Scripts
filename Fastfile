# Fastlane Documentation:
# https://github.com/fastlane/fastlane/tree/master/fastlane/docs
#
# Fastlane actions: 
# https://docs.fastlane.tools/actions/
#
# Bitrise with fastlane example:
# https://github.com/bitrise-samples/fastlane/blob/master/BitriseFastlaneSample/fastlane/Fastfile
#
# AUTHENTICATION
# https://docs.fastlane.tools/best-practices/continuous-integration/
# https://github.com/fastlane/fastlane/tree/master/spaceship
#
# Change the syntax highlighting to Ruby
# All lines starting with a # are ignored when running `fastlane`
#
#
# This file was created by the help of https://gist.github.com/ulhas/e8e529d08849b8cda947
#
#
# NOTE: To use faslane first do:
#           gem install bundler && bundle install
#
#       and then run:
#           bundle exec fastlane release product_name:Gasoline version:"1.0.0" github_account:"blackjacx"
#           bundle exec fastlane test product_name:Gasoline
#
# Configure the following environment variables for your project 
# on your CI and in fastlane/.env:
#
# ENV["GITHUB_TOKEN"] = "..."
# ENV["SLACK_URL"] = "..."
# ENV["BITRISE_DEPLOY_DIR"] = "..."
# ENV['CRASHLYTICS_API_TOKEN'] = "..."
# ENV['CRASHLYTICS_BUILD_SECRET'] = "..."
# ENV['FASTLANE_USER'] = "..."
# ENV['FASTLANE_PASSWORD'] = "..."
# ENV['FASTLANE_DONT_STORE_PASSWORD'] = "..."



# This is the minimum version number required. 
# Update this, if you use features of a newer version.
fastlane_version "2.55.0"

default_platform :ios


platform :ios do

  before_all do
  end

  after_all do
    # This is called, only if the executed lane was successful
    slack(success: true, message: "*Successfully Built App 🎉*")
  end

  error do |lane, exception|
    # This block is called, only if the executed lane failed
    slack(success: false, message: "*#{lane} failed with #{exception.message}*")
  end


  ##############################################################################
  # Playground Lane
  ##############################################################################

  lane :playground do |options|
    stamp_changelog(
      section_identifier: version, # Stamp Unreleased section with newly released build number
      placeholder_line: "* Your contribution here."
    )
  end


  ##############################################################################
  # Preparing Release
  ##############################################################################

  lane :prepare_release do |options|

    raise "No product_name provided!".red unless options[:product_name]
    raise "No version provided!".red unless options[:version]

    product_name = options[:product_name]
    version = options[:version]

    ensure_git_status_clean
    ensure_git_branch(branch: "develop")

    increment_version_number(
      version_number: version
    )

    stamp_changelog(
      section_identifier: version, # Stamp Unreleased section with newly released build number
      placeholder_line: "* Your contribution here."
    )
  end


  ##############################################################################
  # Release Lane
  ##############################################################################

  desc "Releases a new product version"
  lane :release do |options|

    possible_tester_groups = ["appstore", "beta", "nightly"]
    tester_groups = "#{options[:tester_groups]}"

    raise "No product_name provided!".red unless options[:product_name]
    raise "No build provided!".red unless options[:build]
    raise "No github_account provided!".red unless options[:github_account]
    raise "No tester groups provided! Provide at least one of the following as ruby array: #{possible_tester_groups}".red if options[:tester_groups].empty?
    
    product_name = options[:product_name]
    build = options[:build]
    github_account = options[:github_account]
    project = "#{product_name}.xcodeproj"
    workspace = "#{product_name}.xcworkspace"
    version = get_version_number(xcodeproj: "#{project}", target: "#{product_name}")
    deploy_dir = ENV["BITRISE_DEPLOY_DIR"]
    ipa_name = "#{product_name}.ipa"
    ipa_path = "#{deploy_dir}/#{product_name}.ipa"
    dsym_path = "#{deploy_dir}/#{product_name}.app.dSYM.zip"
    run_danger = options[:run_danger]
    tester_whatsnew = File.read("./metadata/whatsnew.txt")
    export_options_plist = "./fastlane/ExportOptions.plist"

    test(product_name: product_name, run_danger: run_danger)

    # Increments CFBundleVersion by one and sets CFBundleShortVersionString
    increment_version_number(version_number: version)
    increment_build_number(build_number: build)

    # read changelog
    changelog = read_changelog(
      section_identifier: "[#{version}]",
      excluded_markdown_elements: []
    )

    changelog_no_md = changelog
        .gsub(/\* \[(.+): /, '* ')
        .gsub(/ - (.+)\./, '')
        .gsub(/###/, '')
        .strip

    gym(
      workspace: workspace, 
      scheme: product_name,
      buildlog_path: deploy_dir,
      output_directory: deploy_dir,
      output_name: ipa_name,
      export_options: export_options_plist,
      include_bitcode: true,
      include_symbols: true,
      clean: true
    )

    set_github_release(
      repository_name: "#{github_account}/#{product_name}",
      api_token: ENV["GITHUB_TOKEN"],
      name: version,
      tag_name: version,
      description: (changelog || "No changelog provided"),
      commitish: "master"
    )
    
    crashlytics(
      crashlytics_path: "./Pods/Crashlytics/iOS/Crashlytics.framework",
      groups: tester_groups,
      notes: changelog_no_md,
      notifications: false
    )

    pilot(
      changelog: changelog_no_md,
      ipa: "#{ipa_path}",
      groups: tester_groups,
      distribute_external: true
    )

    refresh_dsyms

    sh("ls -la #{deploy_dir}")
    puts tester_groups
  end


  ##############################################################################
  # Test Lane
  ##############################################################################

  desc "Runs tests optionally with danger"
  lane :test do |options|

    raise "No product_name provided!".red unless options[:product_name]

    product_name = options[:product_name]
    run_danger = options[:run_danger]

    # install all dependencies
    cocoapods(use_bundle_exec: true, repo_update: true)
    
    # Easily run tests of your iOS app using scan
    scan(
      workspace: "#{product_name}.xcworkspace",
      scheme: product_name,
      clean: true,
      devices: ["iPhone 8"],
      skip_build: false,
      output_types: "html",
      skip_slack: true,
      output_directory: ENV["BITRISE_DEPLOY_DIR"]
    )

    # Runs danger for the project
    if run_danger
      danger(
        use_bundle_exec: true,
        github_api_token: ENV["GITHUB_TOKEN"],
        verbose: true
      )
    end
  end


  ##############################################################################
  # Manually Refreshing Crashlytics dSYM's
  ##############################################################################

  lane :refresh_dsyms do
    # This needs to be done after uploading a new app to ITC since Apple 
    # recompiles the apps from bitcode hich results in new dSYM's.
    # NOTE: upload_symbols_to_crashlytics relies in the installed Fabric.app
    # https://github.com/fastlane/fastlane/issues/10255
    # https://krausefx.com/blog/download-dsym-symbolication-files-from-itunes-connect-for-bitcode-ios-apps
    # https://docs.fabric.io/apple/crashlytics/missing-dsyms.html#upload-symbols-script
    user = ENV["FASTLANE_USER"]
    app_id = CredentialsManager::AppfileConfig.try_fetch_value(:app_identifier)
    
    download_dsyms(username: user, app_identifier: app_id)
    upload_symbols_to_crashlytics
  end


  ##############################################################################
  # Increment build number based on latest testflight buuild number by one.
  ##############################################################################

  desc "Increment build number based on latest testflight buuild number by one."
  lane :increment_latest_build_number do
    check_environment

    itc_team_id = ENV["FASTLANE_ITC_TEAM_ID"]
    bundle_id = ENV["APP_BUNDLE_ID"]
    version = options[:version]
    UI.crash!("No version provided") if version.nil? || version.empty?

    latest_testflight_build_number(
      team_id: itc_team_id,
      app_identifier:  bundle_id,
      initial_build_number: 0,
      version: version
    )
  end


  #############################################################################
  # Check for parameters that have to be set in each environment
  #############################################################################

  desc "Check for parameters that have to be set in each environment."
  private_lane :check_environment do |options|
    crash_if_nil("FASTLANE_ITC_TEAM_ID")
    crash_if_nil("APP_APPLE_TEAM_ID")
    crash_if_nil("APP_APPLE_TEAM_NAME")
    crash_if_nil("APP_APP_BUNDLE_ID")
    crash_if_nil("APP_SCHEME")
    crash_if_nil("APP_TARGET")
    crash_if_nil("APP_TESTER_GROUPS")
  end
end
