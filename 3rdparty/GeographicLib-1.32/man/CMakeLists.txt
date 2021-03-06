# The man pages are maintained as .pod (plain old documentatoin) files.
# In maintainer mode, there are used to create real man pages (extension
# .1), usage files (extension .usage) for including in the tool itself,
# and html versions of the man pages (extension .1.html) for use from
# the doxygen generated documentation

# Only the maintainer tries to generate the derived files and the .usage
# files are in the build tree.  For non-maintainers, the .usages files
# are in the source tree.
if (MAINTAINER)
  add_custom_target (distrib-man)
  add_custom_target (man ALL)
endif ()

set (MANPAGES)
set (USAGE)
set (HTMLMAN)

# Loop over the tools building up lists of the derived files.  Also in
# maintainer mode, specify how the derived files are created.  The sed
# replacements for the .1.html files glue in a style sheet and implement
# cross-referencing between the tools.  The .usage files are generated
# by a shell script makeusage.sh.
foreach (TOOL ${TOOLS})
  set (MANPAGES ${MANPAGES} ${CMAKE_CURRENT_BINARY_DIR}/${TOOL}.1)
  set (USAGE ${USAGE} ${CMAKE_CURRENT_BINARY_DIR}/${TOOL}.usage)
  set (HTMLMAN ${HTMLMAN} ${CMAKE_CURRENT_BINARY_DIR}/${TOOL}.1.html)
  if (MAINTAINER)
    add_custom_command (OUTPUT ${TOOL}.1
      COMMAND pod2man --center=\"GeographicLib Utilities\"
        --release=\"GeographicLib ${GeographicLib_VERSION}\"
        ${CMAKE_CURRENT_SOURCE_DIR}/${TOOL}.pod > ${TOOL}.1
      COMMENT "Building man page for ${TOOL}"
      MAIN_DEPENDENCY ${TOOL}.pod)
    add_custom_command (OUTPUT ${TOOL}.1.html
      COMMAND
        pod2html --noindex ${CMAKE_CURRENT_SOURCE_DIR}/${TOOL}.pod |
        sed -e 's%<head>%<head><link href="http://search.cpan.org/s/style.css" rel="stylesheet" type="text/css">%'
          -e 's%<code>\\\([^<>]*\\\)\(\\\(.\\\)\)</code>%<a href="\\1.\\2.html">&</a>%'g > ${TOOL}.1.html &&
        cp ${TOOL}.1.html ../doc/html-stage/
      COMMENT "Building html version of man page for ${TOOL}"
      MAIN_DEPENDENCY ${TOOL}.pod)
    add_custom_command (OUTPUT ${TOOL}.usage
      COMMAND sh ${CMAKE_CURRENT_SOURCE_DIR}/makeusage.sh
        ${CMAKE_CURRENT_SOURCE_DIR}/${TOOL}.pod > ${TOOL}.usage
      COMMENT "Building usage code for ${TOOL}"
      MAIN_DEPENDENCY ${TOOL}.pod)
  else ()
    # Replace configure_file (... COPYONLY) with file (COPY ...)  once
    # cmake version 2.8 is required
    if (EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/${TOOL}.usage)
      configure_file (${TOOL}.usage ${CMAKE_CURRENT_BINARY_DIR}/${TOOL}.usage
        COPYONLY)
    else ()
      configure_file (dummy.usage.in ${CMAKE_CURRENT_BINARY_DIR}/${TOOL}.usage
        @ONLY)
    endif ()
    if (EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/${TOOL}.1)
      configure_file (${TOOL}.1 ${CMAKE_CURRENT_BINARY_DIR}/${TOOL}.1
        COPYONLY)
    else ()
      configure_file (dummy.1.in ${CMAKE_CURRENT_BINARY_DIR}/${TOOL}.1
        @ONLY)
    endif ()
    if (EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/${TOOL}.1.html)
      configure_file (${TOOL}.1.html ${CMAKE_CURRENT_BINARY_DIR}/${TOOL}.1.html
        COPYONLY)
    else ()
      configure_file (dummy.1.html.in ${CMAKE_CURRENT_BINARY_DIR}/${TOOL}.1.html
        @ONLY)
    endif ()
    if (DOXYGEN_FOUND)
      file (COPY ${CMAKE_CURRENT_BINARY_DIR}/${TOOL}.1.html
        DESTINATION ../doc/html-stage)
    endif ()
  endif ()
endforeach ()

# Add the extra maintainer tasks into the dependency list.  The
# distrib-man target copies the derived documentation files into the
# source tree.
if (MAINTAINER)
  add_custom_target (manpages ALL DEPENDS ${MANPAGES}
    COMMENT "Building all the man pages")
  add_custom_target (usage ALL DEPENDS ${USAGE}
    COMMENT "Converting the man pages to online usage")
  add_custom_target (htmlman ALL DEPENDS ${HTMLMAN}
    COMMENT "Building html versions of the man pages")
  add_dependencies (man manpages usage htmlman)
  add_dependencies (distrib-man man)
  add_custom_command (TARGET distrib-man
    COMMAND
      for f in ${MANPAGES} ${USAGE} ${HTMLMAN}\; do
        cmp "$$f" ${CMAKE_CURRENT_SOURCE_DIR}/`basename "$$f"` >/dev/null 2>&1||
        install -m 644 "$$f" ${CMAKE_CURRENT_SOURCE_DIR}\; done
    COMMENT "Installing man documentation page in source tree")
endif ()

# Install the man pages.
install (FILES ${MANPAGES} DESTINATION share/man/man1)
