
outdir =  ../../../../output
pyoutdir = ${outdir}/py
javaoutdir = ${outdir}/java

protoc = /Users/markd/compbio/ga4gh/src/protobuf/protobuf-3.0.0-alpha-3/src/.libs/protoc

all: ${pyoutdir}/DONE

${pyoutdir}/DONE: protocol.proto
	@mkdir -p $(dir $@)
	${protoc} --python_out=${pyoutdir} $<

