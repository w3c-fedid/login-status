.PHONY: all spec idl clean

all: spec idl

spec: index.html

idl: login-status.idl

login-status.idl: index.bs
	./extract_idl.rb login-status.idl

index.html: index.bs
	bikeshed --die-on=fatal spec index.bs

clean:
	rm index.html
	rm login-status.idl
