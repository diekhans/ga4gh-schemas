
all: protocol.pdf

protocol.pdf: protocol.md
	pandoc -f markdown_github  protocol.md  -o protocol.pdf
	open protocol.pdf

savebak:
	savebak -r ${hgwdev} protocol.md protocol.md .emacs.bak ../src/main/resources/protobuf
