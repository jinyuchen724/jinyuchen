# linux排查问题命令

 
#####查看每个线程cpu占用

`ps -mp pid -o THREAD,tid,time`

#####查看进程堆栈信息
`jstack -l pid > /xx`



kubeless function deploy exp --runtime nodejs8 --from-file exp.js --handler exp.init --dependencies package.json --env NPM_REGISTRY=http://npm.idcvdian.com

kubeless function delete exp

kubeless function call exp

