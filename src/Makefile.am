tool = mytool

binexedir = $(prefix)/binexe
paramdir = $(prefix)/param
ahelpdir = $(prefix)/share/doc/xml

binexe_PROGRAMS = $(tool)

mytool_SOURCES = pars2tab.cc
mytool_CPPFLAGS = $(CIAO_CFLAGS)
mytool_LDADD = $(CIAO_LIBS) 

dist_param_DATA = $(tool).par
dist_ahelp_DATA = $(tool).xml

install-data-hook:
	chmod 444 $(paramdir)/$(dist_param_DATA)
	chmod 444 $(ahelpdir)/$(dist_ahelp_DATA)
