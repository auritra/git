SHELL := /bin/bash
SUDO := sudo
C_INCLUDE_PATH := /usr/include
CPLUS_INCLUDE_PATH := /usr/include
LD_LIBRARY_PATH := /usr/lib

OSX_VERSION := $(shell sw_vers -productVersion)
TARGET_FLAGS := -mmacosx-version-min=$(OSX_VERSION) -DMACOSX_DEPLOYMENT_TARGET=$(OSX_VERSION)

uname_M := $(shell sh -c 'uname -m 2>/dev/null || echo not')

ARCH_UNIV := universal
ARCH_FLAGS := -arch x86_64 -arch arm64

CFLAGS := $(TARGET_FLAGS) $(ARCH_FLAGS)
LDFLAGS := $(TARGET_FLAGS) $(ARCH_FLAGS)

PREFIX := /usr/local
GIT_PREFIX := $(PREFIX)/git

BUILD_DIR := $(GITHUB_WORKSPACE)/payload
DESTDIR := $(PWD)/stage/git-$(ARCH_UNIV)-$(VERSION)
ARTIFACTDIR := build-artifacts
SUBMAKE := $(MAKE) C_INCLUDE_PATH="$(C_INCLUDE_PATH)" CPLUS_INCLUDE_PATH="$(CPLUS_INCLUDE_PATH)" LD_LIBRARY_PATH="$(LD_LIBRARY_PATH)" TARGET_FLAGS="$(TARGET_FLAGS)" CFLAGS="$(CFLAGS)" LDFLAGS="$(LDFLAGS)" NO_GETTEXT=1 NO_DARWIN_PORTS=1 prefix=$(GIT_PREFIX) GIT_BUILT_FROM_COMMIT="$(GIT_BUILT_FROM_COMMIT)" DESTDIR=$(DESTDIR)
CORES := $(shell bash -c "sysctl hw.ncpu | awk '{print \$$2}'")

# Guard against environment variables
APPLE_APP_IDENTITY =
APPLE_INSTALLER_IDENTITY =
APPLE_KEYCHAIN_PROFILE =

.PHONY: image pkg payload codesign notarize

.SECONDARY:

$(DESTDIR)$(GIT_PREFIX)/VERSION-$(VERSION)-$(ARCH_UNIV):
	rm -f $(BUILD_DIR)/git-$(VERSION)/osx-installed*
	mkdir -p $(DESTDIR)$(GIT_PREFIX)
	touch $@

$(BUILD_DIR)/git-$(VERSION)/osx-built-keychain:
	cd $(BUILD_DIR)/git-$(VERSION)/contrib/credential/osxkeychain; $(SUBMAKE) CFLAGS="$(CFLAGS) -g -O2 -Wall"
	touch $@

$(BUILD_DIR)/git-$(VERSION)/osx-built:
	[ -d $(DESTDIR)$(GIT_PREFIX) ] && $(SUDO) rm -rf $(DESTDIR) || echo ok
	cd $(BUILD_DIR)/git-$(VERSION); $(SUBMAKE) -j $(CORES) all strip
	touch $@

$(BUILD_DIR)/git-$(VERSION)/osx-installed-bin: $(BUILD_DIR)/git-$(VERSION)/osx-built $(BUILD_DIR)/git-$(VERSION)/osx-built-keychain
	cd $(BUILD_DIR)/git-$(VERSION); $(SUBMAKE) install
	cp $(BUILD_DIR)/git-$(VERSION)/contrib/credential/osxkeychain/git-credential-osxkeychain $(DESTDIR)$(GIT_PREFIX)/bin/git-credential-osxkeychain
	mkdir -p $(DESTDIR)$(GIT_PREFIX)/contrib/completion
	cp $(BUILD_DIR)/git-$(VERSION)/contrib/completion/git-completion.bash $(DESTDIR)$(GIT_PREFIX)/contrib/completion/
	cp $(BUILD_DIR)/git-$(VERSION)/contrib/completion/git-completion.zsh $(DESTDIR)$(GIT_PREFIX)/contrib/completion/
	cp $(BUILD_DIR)/git-$(VERSION)/contrib/completion/git-prompt.sh $(DESTDIR)$(GIT_PREFIX)/contrib/completion/
	# This is needed for Git-Gui, GitK
	mkdir -p $(DESTDIR)$(GIT_PREFIX)/lib/perl5/site_perl
	[ ! -f $(DESTDIR)$(GIT_PREFIX)/lib/perl5/site_perl/Error.pm ] && cp $(BUILD_DIR)/git-$(VERSION)/perl/private-Error.pm $(DESTDIR)$(GIT_PREFIX)/lib/perl5/site_perl/Error.pm || echo done
	touch $@

$(BUILD_DIR)/git-$(VERSION)/osx-installed-man: $(BUILD_DIR)/git-$(VERSION)/osx-installed-bin
	mkdir -p $(DESTDIR)$(GIT_PREFIX)/share/man
	cp -R $(GITHUB_WORKSPACE)/manpages/ $(DESTDIR)$(GIT_PREFIX)/share/man
	touch $@

$(BUILD_DIR)/git-$(VERSION)/osx-built-subtree:
	cd $(BUILD_DIR)/git-$(VERSION)/contrib/subtree; $(SUBMAKE) XML_CATALOG_FILES="$(XML_CATALOG_FILES)" all git-subtree.1
	touch $@

$(BUILD_DIR)/git-$(VERSION)/osx-installed-subtree: $(BUILD_DIR)/git-$(VERSION)/osx-built-subtree
	mkdir -p $(DESTDIR)
	cd $(BUILD_DIR)/git-$(VERSION)/contrib/subtree; $(SUBMAKE) XML_CATALOG_FILES="$(XML_CATALOG_FILES)" install install-man
	touch $@

$(BUILD_DIR)/git-$(VERSION)/osx-installed-assets: $(BUILD_DIR)/git-$(VERSION)/osx-installed-bin
	mkdir -p $(DESTDIR)$(GIT_PREFIX)/etc
	cat assets/etc/gitconfig.osxkeychain >> $(DESTDIR)$(GIT_PREFIX)/etc/gitconfig
	cp assets/uninstall.sh $(DESTDIR)$(GIT_PREFIX)/uninstall.sh
	sh -c "echo .DS_Store >> $(DESTDIR)$(GIT_PREFIX)/share/git-core/templates/info/exclude"

symlinks:
	mkdir -p $(ARTIFACTDIR)$(PREFIX)/bin
	cd $(ARTIFACTDIR)$(PREFIX)/bin; find ../git/bin -type f -exec ln -sf {} \;
	for man in man1 man3 man5 man7; do mkdir -p $(ARTIFACTDIR)$(PREFIX)/share/man/$$man; (cd $(ARTIFACTDIR)$(PREFIX)/share/man/$$man; ln -sf ../../../git/share/man/$$man/* ./); done
	ruby ../scripts/symlink-git-hardlinks.rb $(ARTIFACTDIR)
	touch $@

$(BUILD_DIR)/git-$(VERSION)/osx-installed: $(DESTDIR)$(GIT_PREFIX)/VERSION-$(VERSION)-$(ARCH_UNIV) $(BUILD_DIR)/git-$(VERSION)/osx-installed-man $(BUILD_DIR)/git-$(VERSION)/osx-installed-assets $(BUILD_DIR)/git-$(VERSION)/osx-installed-subtree
	find $(DESTDIR)$(GIT_PREFIX) -type d -exec chmod ugo+rx {} \;
	find $(DESTDIR)$(GIT_PREFIX) -type f -exec chmod ugo+r {} \;
	touch $@

$(BUILD_DIR)/git-$(VERSION)/osx-built-assert-$(ARCH_UNIV): $(BUILD_DIR)/git-$(VERSION)/osx-built
	File $(BUILD_DIR)/git-$(VERSION)/git
	File $(BUILD_DIR)/git-$(VERSION)/contrib/credential/osxkeychain/git-credential-osxkeychain
	touch $@

disk-image/VERSION-$(VERSION)-$(ARCH_UNIV):
	rm -f disk-image/*.pkg disk-image/VERSION-* disk-image/.DS_Store
	mkdir disk-image
	touch "$@"

pkg_cmd := pkgbuild --identifier com.git.pkg --version $(VERSION) \
	--root $(ARTIFACTDIR)$(PREFIX) --scripts assets/scripts \
	--install-location $(PREFIX) --component-plist ./assets/git-components.plist

ifdef APPLE_INSTALLER_IDENTITY
	pkg_cmd += --sign "$(APPLE_INSTALLER_IDENTITY)"
endif

pkg_cmd += disk-image/git-$(VERSION)-$(ARCH_UNIV).pkg
disk-image/git-$(VERSION)-$(ARCH_UNIV).pkg: disk-image/VERSION-$(VERSION)-$(ARCH_UNIV) symlinks
	$(pkg_cmd)

git-%-$(ARCH_UNIV).dmg:
	hdiutil create git-$(VERSION)-$(ARCH_UNIV).uncompressed.dmg -fs HFS+ -srcfolder disk-image -volname "Git $(VERSION) $(ARCH_UNIV)" -ov 2>&1 | tee err || { \
		grep "Resource busy" err && \
		sleep 5 && \
		hdiutil create git-$(VERSION)-$(ARCH_UNIV).uncompressed.dmg -fs HFS+ -srcfolder disk-image -volname "Git $(VERSION) $(ARCH_UNIV)" -ov; }
	hdiutil convert -format UDZO -o $@ git-$(VERSION)-$(ARCH_UNIV).uncompressed.dmg
	rm -f git-$(VERSION)-$(ARCH_UNIV).uncompressed.dmg

payload: $(BUILD_DIR)/git-$(VERSION)/osx-installed $(BUILD_DIR)/git-$(VERSION)/osx-built-assert-$(ARCH_UNIV)

pkg: disk-image/git-$(VERSION)-$(ARCH_UNIV).pkg

image: git-$(VERSION)-$(ARCH_UNIV).dmg

ifdef APPLE_APP_IDENTITY
codesign:
	@$(CURDIR)/../scripts/codesign.sh --payload="build-artifacts/usr/local/git" \
		--identity="$(APPLE_APP_IDENTITY)" \
		--entitlements="$(CURDIR)/entitlements.xml"
endif

# Notarization can only happen if the package is fully signed
ifdef APPLE_KEYCHAIN_PROFILE
notarize:
	@$(CURDIR)/../scripts/notarize.sh \
		--package="disk-image/git-$(VERSION)-$(ARCH_UNIV).pkg" \
		--keychain-profile="$(APPLE_KEYCHAIN_PROFILE)"
endif
