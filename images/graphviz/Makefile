all: generate


generate: 
	@for fn in $$(ls  *.dot ) ; do dot -Tpng $$fn > $$fn.png ; done 

clean:
	rm -rf *.png
