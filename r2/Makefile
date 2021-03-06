# The contents of this file are subject to the Common Public Attribution
# License Version 1.0. (the "License"); you may not use this file except in
# compliance with the License. You may obtain a copy of the License at
# http://code.reddit.com/LICENSE. The License is based on the Mozilla Public
# License Version 1.1, but Sections 14 and 15 have been added to cover use of
# software over a computer network and provide for limited attribution for the
# Original Developer. In addition, Exhibit A has been modified to be consistent
# with Exhibit B.
#
# Software distributed under the License is distributed on an "AS IS" basis,
# WITHOUT WARRANTY OF ANY KIND, either express or implied. See the License for
# the specific language governing rights and limitations under the License.
#
# The Original Code is reddit.
#
# The Original Developer is the Initial Developer.  The Initial Developer of
# the Original Code is reddit Inc.
#
# All portions of the code written by reddit are Copyright (c) 2006-2012 reddit
# Inc. All Rights Reserved.
###############################################################################

SHELL=/bin/sh
PYTHON=python
SED=sed
CAT=cat
BUILD_DIR=build

.PHONY: clean
all: pyx static ini
clean: clean_pyx clean_i18n clean_static

DEFS_PY := Makefile.py
DEFS_FILE := Makefile.defs
.PHONY: $(DEFS_FILE)
$(shell $(PYTHON) $(DEFS_PY) > $(DEFS_FILE))
include $(DEFS_FILE)
ifdef DEFS_SUCCESS
$(info [+] including definitions from $(DEFS_PY))
else
$(error $(DEFS_PY) failed. aborting)
endif

#################### Cython
PYX_FILES := $(shell find . -name \*.pyx)
PYX_C_FILES := $(PYX_FILES:.pyx=.c)
PYX_SO_FILES := $(PYX_FILES:.pyx=.so)
PYX_BUILDSTAMP := $(BUILD_DIR)/pyx-buildstamp

.PHONY: clean_pyx

pyx: $(PYX_BUILDSTAMP)

$(PYX_C_FILES): %.c: %.pyx
	cython $<

# this won't rebuild pyx if you're a jerk and delete a .so file without deleting the buildstamp
# make clean && make pyx will fix this
$(PYX_BUILDSTAMP): $(PYX_C_FILES)
	$(PYTHON) setup.py build_ext --inplace
	touch $@

clean_pyx:
	rm -f $(PYX_BUILDSTAMP) $(PYX_C_FILES) $(PYX_SO_FILES)

#################### i18n
STRINGS_FILE := r2/lib/strings.py
GENERATED_STRINGS_FILE := r2/lib/generated_strings.py

.PHONY: i18n clean_i18n

# POTFILE is set by Makefile.py
i18n: $(GENERATED_STRINGS_FILE)
	$(PYTHON) setup.py extract_messages -o $(POTFILE)

$(GENERATED_STRINGS_FILE): $(STRINGS_FILE)
	paster run standalone $(STRINGS_FILE) -c "generate_strings()" > $(GENERATED_STRINGS_FILE)

clean_i18n:
	rm -f $(GENERATED_STRINGS_FILE)

#################### ini files
UPDATE_FILES := $(wildcard *.update)
INIFILES := $(UPDATE_FILES:.update=.ini)

.PHONY: clean_ini

ini: $(INIFILES)

$(INIFILES): %.ini: %.update example.ini
	  ./updateini.py example.ini $< > $@ || rm $@ 

clean_ini:
	rm $(INIFILES)

#################### CSS file lists
SPRITED_STYLESHEETS += reddit.css compact.css
LESS_STYLESHEETS := goldinfo.less
OTHER_STYLESHEETS := reddit-ie6-hax.css reddit-ie7-hax.css mobile.css highlight.css

#################### Static Files
STATIC_ROOT := r2/public
STATIC_FILES := $(shell find $(STATIC_ROOT) -readable)
STATIC_BUILD_ROOT := $(BUILD_DIR)/public
STATIC_BUILD_DIR := $(STATIC_BUILD_ROOT)/static
STATIC_BUILDSTAMP := $(BUILD_DIR)/static-buildstamp

.PHONY: clean_static

static: plugin_static pyx css js names

clean_static:
	rm -rf $(STATIC_BUILDSTAMP) $(STATIC_BUILD_DIR) $(PLUGIN_BUILDSTAMPS) $(MANGLE_BUILDSTAMP)

$(STATIC_BUILDSTAMP): $(STATIC_FILES)
	cp -ruTL $(STATIC_ROOT) $(STATIC_BUILD_ROOT)
	touch $@

#################### Plugin static files
PLUGIN_BUILDSTAMPS := $(foreach plugin,$(PLUGINS),$(BUILD_DIR)/plugin-$(plugin)-buildstamp)

plugin_static: $(PLUGIN_BUILDSTAMPS)

define PLUGIN_STATIC_TEMPLATE
$(BUILD_DIR)/plugin-$(1)-buildstamp: $(STATIC_BUILDSTAMP) $(shell find $(PLUGIN_PATH_$(1))/public)
	if [ -f $(PLUGIN_PATH_$(1))/../Makefile ]; then \
		$(MAKE) -C $(PLUGIN_PATH_$(1))/../ static; \
	fi
	cp -r --preserve=timestamps $(PLUGIN_PATH_$(1))/public/* $(STATIC_BUILD_ROOT)/
	touch $$@

$(info [+] adding make rules from plugin "$(1)")
-include $(PLUGIN_PATH_$(1))/../Makefile.plugin
endef
$(foreach plugin,$(PLUGINS),$(eval $(call PLUGIN_STATIC_TEMPLATE,$(plugin))))

#### Stylesheets
LESSC := r2/lib/contrib/less.js/bin/lessc
CSS_COMPRESS := $(PYTHON) r2/lib/contrib/rcssmin.py
CSS_SOURCE_DIR := $(STATIC_BUILD_DIR)/css

PROCESSED_SPRITED_STYLESHEETS := $(addprefix $(STATIC_BUILD_DIR)/, $(SPRITED_STYLESHEETS))
SPRITES := $(addprefix $(STATIC_BUILD_DIR)/, $(patsubst %.css,sprite-%.png, $(SPRITED_STYLESHEETS)))

LESS_OUTPUTS := $(addprefix $(STATIC_BUILD_DIR)/, $(patsubst %.less,%.css, $(LESS_STYLESHEETS)))

MINIFIED_OTHER_STYLESHEETS := $(addprefix $(STATIC_BUILD_DIR)/, $(OTHER_STYLESHEETS))

PROCESSED_STYLESHEETS := $(PROCESSED_SPRITED_STYLESHEETS) $(MINIFIED_OTHER_STYLESHEETS) $(LESS_OUTPUTS)
RTL_STYLESHEETS := $(PROCESSED_STYLESHEETS:.css=-rtl.css)

CSS_OUTPUTS = $(PROCESSED_STYLESHEETS) $(RTL_STYLESHEETS) $(SPRITES)

.PHONY: clean_css

css: $(STATIC_BUILDSTAMP) $(CSS_OUTPUTS)


$(LESS_OUTPUTS): $(STATIC_BUILD_DIR)/%.css : $(CSS_SOURCE_DIR)/%.less
	rm -f $@
	$(LESSC) $< | $(CSS_COMPRESS) > $@

$(MINIFIED_OTHER_STYLESHEETS): $(STATIC_BUILD_DIR)/%.css: $(CSS_SOURCE_DIR)/%.css
	# when static file names are mangled, the original becomes a symlink to the mangled name
	# remove the original file here in case it's a symlink so we don't just rewrite the old file
	rm -f $@
	$(CAT) $< | $(CSS_COMPRESS) > $@

$(STATIC_BUILD_DIR)/sprite-%.png $(STATIC_BUILD_DIR)/%.css: $(CSS_SOURCE_DIR)/%.css $(STATIC_BUILDSTAMP)
	# see above
	rm -f $(STATIC_BUILD_DIR)/sprite-$*.png $(STATIC_BUILD_DIR)/$*.css
	$(PYTHON) r2/lib/nymph.py $< $(STATIC_BUILD_DIR)/sprite-$*.png | $(CSS_COMPRESS) > $(STATIC_BUILD_DIR)/$*.css

$(RTL_STYLESHEETS): %-rtl.css : %.css
	# see above
	rm -f $@
	$(SED) -e "s/left/>####</g" \
	   -e "s/right/left/g" \
	   -e "s/>####</right/g" \
	   -e "s/\(margin\|padding\):\s*\([^; ]\+\)\s\+\([^; ]\+\)\s\+\([^; ]\+\)\s\+\([^; ]\+\)/\1:\2 \5 \4 \3/g" $< > $@

clean_css:
	rm -f $(CSS_OUTPUTS)

#### JS

.PHONY: clean_js

js: $(STATIC_BUILDSTAMP) $(JS_OUTPUTS)

define JS_MODULE_TEMPLATE
$(JS_MODULE_OUTPUTS_$(1)): $(JS_MODULE_DEPS_$(1))
	# remove mangled output symlinks, similar to above.
	rm -f $(JS_MODULE_OUTPUTS_$(1))
	paster run standalone r2/lib/js.py -c "build_module('$(1)')"
	touch $$@
endef

# apply the module template to each of the modules
# so they source their deps from js.py and build accordingly
$(foreach module,$(JS_MODULES),$(eval $(call JS_MODULE_TEMPLATE,$(module))))

clean_js:
	rm -f $(STATIC_BUILD_DIR)/*.js

#### name mangling
MANGLEABLE_FILES := $(CSS_OUTPUTS) $(JS_OUTPUTS)
MANGLE_BUILDSTAMP := $(BUILD_DIR)/mangle-buildstamp
NAMES_FILE := $(STATIC_BUILD_DIR)/names.json
MANGLED_FILES := $(wildcard $(foreach file,$(MANGLEABLE_FILES),$(basename $(file)).*$(suffix $(file))))

.PHONY: clean_names

names: $(MANGLE_BUILDSTAMP)

$(MANGLE_BUILDSTAMP): $(MANGLEABLE_FILES)
	$(PYTHON) r2/lib/static.py $(NAMES_FILE) $(MANGLEABLE_FILES)
	touch $@

clean_names:
	rm -f $(MANGLE_BUILDSTAMP) $(NAMES_FILES) $(MANGLEABLE_FILES) $(MANGLED_FILES)

#### gzip!
GZIPPABLE := $(filter %.css %.js,$(MANGLED_FILES))
GZIPPED := $(addsuffix .gzip,$(GZIPPABLE))

.PHONY: clean_gzip

gzip: $(GZIPPED)

$(GZIPPED): %.gzip: %
	gzip -c $< > $@

clean_gzip:
	rm -f $(GZIPPED)

$(shell rm -f $(DEFS_FILE))
