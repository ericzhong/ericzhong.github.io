

## VirtualEnv

Install

	sudo pip install virtualenv

Init Dir

	cd DIR
	virtualenv .
	
Activate Env

	source bin/activate
	which python


## PrettyTable

Install

	pip install prettytable

Example Code

	#!/usr/bin/python
	
	from prettytable import PrettyTable
	
	x = PrettyTable(["City name", "Area", "Population", "Annual Rainfall"])
	x.align["City name"] = "l" # Left align city names
	x.padding_width = 1 # One space between column edges and contents (default)
	x.add_row(["Adelaide",1295, 1158259, 600.5])
	x.add_row(["Brisbane",5905, 1857594, 1146.4])
	x.add_row(["Darwin", 112, 120900, 1714.7])
	x.add_row(["Hobart", 1357, 205556, 619.5])
	x.add_row(["Sydney", 2058, 4336374, 1214.8])
	x.add_row(["Melbourne", 1566, 3806092, 646.9])
	x.add_row(["Perth", 5386, 1554769, 869.4])
	print x

Example Output

	+-----------+------+------------+-----------------+
	| City name | Area | Population | Annual Rainfall |
	+-----------+------+------------+-----------------+
	| Adelaide  | 1295 |  1158259   |      600.5      |
	| Brisbane  | 5905 |  1857594   |      1146.4     |
	| Darwin    | 112  |   120900   |      1714.7     |
	| Hobart    | 1357 |   205556   |      619.5      |
	| Sydney    | 2058 |  4336374   |      1214.8     |
	| Melbourne | 1566 |  3806092   |      646.9      |
	| Perth     | 5386 |  1554769   |      869.4      |
	+-----------+------+------------+-----------------+


## ArgParse

###术语

以`PROG [-h] [--foo FOO] [bar [bar …]]`为例

* `option` - '--foo'
* `attribute` - 'foo'
* `optional arguments` - 'FOO'
* `position arguments` - 'bar'

###ArgumentParser()

	class argparse.ArgumentParser(prog=None, usage=None, description=None, epilog=None, parents=[], 
									formatter_class=argparse.HelpFormatter, prefix_chars='-', fromfile_prefix_chars=None,
									argument_default=None, conflict_handler='error', add_help=True)

* `prog` - 指定程序名称 (default: sys.argv[0])
* `usage` - 指定Usage字符串 (default: 参数清单)
* `description` - argument help之前的一段描述文字 (default: none)
* `epilog` - argument help之后的一段描述文字 (default: none)
* `parents` - 继承其它ArgumentParser对象的选项，可以多个（List）
* `formatter_class` - 定制Help输出（通过几个可选的内置方法）
* `prefix_chars` - 自定义optional arguments的前缀，可多个 (default: ‘-‘)
* `fromfile_prefix_chars` - 当从指定文件读取参数，指定文件名时加的前缀 (default: None)
* `argument_default` - 全局默认参数 (default: None)
* `conflict_handler` - 是否解决重复定义冲突，可用于覆盖前面的定义 (usually unnecessary)
* `add_help` - 是否显示“-h, --help  show this help message and exit” (default: True)

###add_argument()

	ArgumentParser.add_argument(name or flags...[, action][, nargs][, const][, default][, type][, choices][, required][, help][, metavar][, dest])


* `name or flags` - 多个选项(-f, --foo)或者一个positional argument（多个写成List）
* `action` - 对command-line arguments的处理
	* `store` - （默认）保存参数值
	* `store_const` - 配合`const`参数使用，保存`const`指定的值
	* `store_true` or `store_false` - 当给出选项时保存为`true`or`false`
	* `append` - 将同一个选项的多个参数保存到一个List里面
	* `append_const` - 同上，不过使用`const`参数指定的值
	* `count` - 计算选项重复次数
	* `help` - ？
	* `version` - 配合`version`参数自定义`--version`的输出
	* 自定义
* `nargs` - 可接受的参数数量
	* 整数 - `nargs=2`表示可接受两个，比如`--foo a b` => `foo=['a', 'b']`
	* `?` - 表示只接受一个参数，可配合`default`参数（没有给出该选项）和`const`参数（仅有该选项，但没有参数）使用
	* `*` - 将所有参数都整合到List里面，比如`a b --foo x y --bar 1 2` => `bar=['1', '2'], baz=['a', 'b'], foo=['x', 'y']`
	* `+` - 同`*`，但必须至少给出一个参数，否则会报错
	* `argparse.REMAINDER` - 将剩下的所有参数和选项都整合到一个List里面
* `const` - 配合`action`和`nargs`使用（可用max,sum等内置函数）
* `default` - 配合其它使用，当没有给出某选项时从`default`取值（可用max,sum等内置函数）
* `type` - 指定参数类型，读取后内部会做转换处理
* `choices` - 只能从一组指定值中选择参数，否则会报错
* `required` - 该参数必须有，否则报错
* `help` - 指定参数的help信息，也可禁止输出help信息
* `metavar` - 指定属性的显示名称，内部值仍然由`dest`指定
* `dest`- 指定属性的内部名称；默认是使用第一个长选项(`--`)，如果全部都是短选项（`-`），使用第一个

###parse_args()

	ArgumentParser.parse_args(args=None, namespace=None)
	
默认从`sys.argv`读取参数，创建一个空的`Namespace`对象存储属性

###Ohter
* 子命令
	* `ArgumentParser.add_subparsers()` - 适用于有子命令的程序
* 参数分组
	* `ArgumentParser.add_argument_group(title=None, description=None)` - 将选项分组显示
* 互斥选项
	* `argparse.add_mutually_exclusive_group(required=False)` - 互斥组中的选项不得同时出现，否则报错
* 默认参数
	* `ArgumentParser.set_defaults(**kwargs)` - 设置选项默认值，适用于某些不需要检查的选项；优先级高于`default`参数
	* `ArgumentParser.get_default(dest)` - 获取选项默认值
* Printing Help 
	* `ArgumentParser.print_usage(file=None)`
	* `ArgumentParser.print_help(file=None)`
	* `ArgumentParser.format_usage()`
	* `ArgumentParser.format_help()`
* 局部分析
	* `ArgumentParser.parse_known_args(args=None, namespace=None)` - 分析剩下的没有定义的参数和选项被一起放到一个List中
* 自定义文件分析
	* `ArgumentParser.convert_arg_line_to_args(arg_line)` - 实现该方法，从自定义文件读取选项和参数
* 退出
	* `ArgumentParser.exit(status=0, message=None)`
	* `ArgumentParser.error(message)`
* 从老的`optparse`升级到新的`argparse`（见官方文档）