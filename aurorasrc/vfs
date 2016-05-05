if fs.native then
	return false, "VFS already installed"
end

local function checkArg(argtype, argn, arg)
	if type(arg) ~= argtype then
		error("invalid arg #" .. argn .. " (" .. argtype .. " expected, got " .. type(arg) .. ")", 1)
	end
end

local proxy = {}
local nativefs = fs; _G.fs = {}
local rootfs = nativefs
function fs.native()
	return nativefs
end
function fs.setRoot(_proxy)
	rootfs = _proxy
end
function fs.getRoot()
	return rootfs
end
function fs.cleanPath(path)
	local parts = {}
	for match in string.gmatch(path, "[^/]+") do
		if match == ".." then
			if #parts < 1 then
				error("Invalid path", 1)
			end
			table.remove(parts, #parts)
		else
			table.insert(parts, match)
		end
	end
	return table.concat(parts, "/")
end
function fs.proxy(path, _proxy)
	checkArg("string", 1, path)
	checkArg("table",  2, _proxy)
	path = fs.cleanPath(path)
	if proxy[path] then
		error("proxy already present at path", 1)
	end
	proxy[path] = _proxy
end
function fs.unproxy(path)
	path = fs.cleanPath(path)
	proxy[path] = nil
end
function fs.getProxy(path)
	local p, pathbase = rootfs, ""
	local depth = 0
	for k, v in pairs(proxy) do
		local _depth = 0
		for _ in string.gmatch(k, "[/]+") do
			_depth = _depth + 1
		end
		if (path == k or path:sub(1, #k + 1) == k .. "/") and _depth >= depth then
			p = v
			pathbase = k
			depth = _depth
		end
	end
	return p, pathbase
end
function fs.getProxiedPath(path)
	local str = string.sub(path, #({fs.getProxy(path)})[2] + 1)
	return fs.cleanPath(str)
end
function fs.listProxy()
	local t = {}
	for path in pairs(proxy) do
		table.insert(t, path)
	end
	table.sort(t)
	return t
end
function fs.list(path)
	path = fs.cleanPath(path)
	return fs.getProxy(path).list(fs.getProxiedPath(path))
end
function fs.move(path, target)
	path = fs.cleanPath(path)
	target = fs.cleanPath(target)
	local p1 = fs.getProxy(path)
	local p2 = fs.getProxy(target)
	if p1 ~= p2 then
		if p1.isDir(fs.getProxiedPath(path)) then
			fs.copy(path, target)
			p1.delete(fs.getProxiedPath(path))
			return
		end
		if p2.exists(fs.getProxiedPath(target)) then
			error("File exists", 1)
		end
		if p1.isReadOnly(fs.getProxiedPath(path)) or p2.isReadOnly(fs.getProxiedPath(target)) then
			error("Access denied", 1)
		end
		local f1 = p1.open(fs.getProxiedPath(path), "r")
		local f2 = p2.open(fs.getProxiedPath(path), "w")
		f2.write(f1.readAll())
		f1.close()
		f2.close()
		p1.delete(fs.getProxiedPath(path))
	else
		p1.move(fs.getProxiedPath(path), fs.getProxiedPath(target))
	end
end
function fs.recursiveCopy(p1, p2, pA, pB)
	for i, v in ipairs(p1.list(pA)) do
		if p1.isDir(fs.combine(pA, v)) then
			if not p2.exists(fs.combine(pB, v)) then
				p2.makeDir(fs.combine(pB, v))
			end
			fs.recursiveCopy(p1, p2, fs.combine(pA, v), fs.combine(pB, v))
		else
			local f1 = p1.open(fs.combine(pA, v), "r")
			local f2 = p2.open(fs.combine(pB, v), "w")
			f2.write(f1.readAll())
			f1.close()
			f2.close()
		end
	end
end
function fs.copy(path, target)
	path = fs.cleanPath(path)
	target = fs.cleanPath(target)
	local p1 = fs.getProxy(path)
	local p2 = fs.getProxy(target)
	if p1 ~= p2 then
		if p1.isDir(fs.getProxiedPath(path)) then
			fs.recursiveCopy(p1, p2, fs.getProxiedPath(path), fs.getProxiedPath(target))
			return
		end
		if p2.exists(fs.getProxiedPath(target)) then
			error("File exists", 1)
		end
		if p2.isReadOnly(fs.getProxiedPath(target)) then
			error("Access denied", 1)
		end
		local f1 = p1.open(fs.getProxiedPath(path), "r")
		local f2 = p2.open(fs.getProxiedPath(target), "w")
		f2.write(f1.readAll())
		f1.close()
		f2.close()
	else
		p1.copy(fs.getProxiedPath(path), fs.getProxiedPath(target))
	end
end
function fs.makeDir(path)
	path = fs.cleanPath(path)
	fs.getProxy(path).makeDir(fs.getProxiedPath(path))
end
function fs.delete(path)
	path = fs.cleanPath(path)
	fs.getProxy(path).delete(fs.getProxiedPath(path))
end
function fs.open(path, mode)
	path = fs.cleanPath(path)
	return fs.getProxy(path).open(fs.getProxiedPath(path), mode)
end
function fs.isDir(path)
	path = fs.cleanPath(path)
	return fs.getProxy(path).isDir(fs.getProxiedPath(path))
end
function fs.exists(path)
	path = fs.cleanPath(path)
	return fs.getProxy(path).exists(fs.getProxiedPath(path))
end
function fs.combine(path1, path2)
	return nativefs.combine(path1, path2)
end
function fs.getName(path)
	path = fs.cleanPath(path)
	return nativefs.getName(path)
end
function fs.getDir(path)
	path = fs.cleanPath(path)
	return nativefs.getDir(path)
end
function fs.isReadOnly(path)
	path = fs.cleanPath(path)
	return fs.getProxy(path).isReadOnly(fs.getProxiedPath(path))
end
function fs.find(path) -- cclite copypaste
	path = fs.cleanPath(path)
	local function recurse_spec(results, path, spec)
		local segment = spec:match('([^/]*)'):gsub('/', '')
		local pattern = '^' .. segment:gsub("[%.%[%]%(%)%%%+%-%?%^%$]","%%%1"):gsub("%z","%%z"):gsub("%*","[^/]-") .. '$'
			if fs.isDir(path) then
			for _, file in ipairs(fs.list(path)) do
				if file:match(pattern) then
					local f = fs.combine(path, file)
						if spec == segment then
						table.insert(results, f)
					end
					if fs.isDir(f) then
						recurse_spec(results, f, spec:sub(#segment + 2))
					end
				end
			end
		end
	end
	local results = {}
	recurse_spec(results, '', path)
	table.sort(results)
	return results
end
function fs.getSize(path)
	path = fs.cleanPath(path)
	return fs.getProxy(path).getSize(fs.getProxiedPath(path))
end
fs.complete = nativefs.complete
function fs.getDrive(path)
	path = fs.cleanPath(path)
	if fs.getProxy(path).getDrive then
		return fs.getProxy(path).getDrive(fs.getProxiedPath(path))
	else
		return "???"
	end
end
function fs.getFreeSpace(path)
	path = fs.cleanPath(path)
	if fs.getProxy(path).getFreeSpace then
		return fs.getProxy(path).getFreeSpace(fs.getProxiedPath(path))
	else
		return 2^31 - 1
	end
end
for i in pairs(nativefs) do
	if not fs[i] then
		print("MISSING: " .. i)
	end
end
function fs.appfsProxy(structure)
	local function getFile(path)
		local file = structure
		for part in string.gmatch(path, "[^/]+") do
			if not file.files[part] then
				return nil
			end
			file = file.files[part]
		end
		return file
	end
	local p = {}
	function p.getDrive()
		return "sys"
	end
	function p.getFreeSpace()
		return 0
	end
	function p.exists(path)
		local file = structure
		for part in string.gmatch(path, "[^/]+") do
			if not file.files[part] then
				return false
			end
			file = file.files[part]
		end
		return file ~= nil
	end
	function p.list(path)
		local file = getFile(path)
		if not file then
			error("Not a directory", 1)
		end
		if not file.files then
			error("Not a directory", 1)
		end
		local files = {}
		for f in pairs(file.files) do
			table.insert(files, f)
		end
		table.sort(files)
		return files
	end
	function p.delete(path)
		if structure.noDelete then
			error("Access denied", 1)
		end
	end
	function p.isDir(path)
		if not getFile(path) then
			return false
		end
		return getFile(path).files ~= nil
	end
	function p.isReadOnly(path)
		return getFile(path).ro == true
	end
	function p.open(path, mode)
		local f = getFile(path)
		if mode == "r" then
			local handle = {closed = false, buffer = f.read and f.read() or ""}
			function handle.readLine()
				if handle.closed then
					return
				end
				if #handle.buffer < 1 then
					return nil
				end
				local tmp = ""
				while #handle.buffer > 0 do
					local chr = handle.buffer:sub(1, 1)
					handle.buffer = handle.buffer:sub(2)
					if chr == "\n" then
						return tmp
					else
						tmp = tmp .. chr
					end
				end
				return tmp
			end
			function handle.readAll()
				if handle.closed then
					return
				end
				local str = handle.buffer
				handle.buffer = ""
				return str
			end
			function handle.close()
				handle.closed = true
			end
			return handle
		elseif mode == "w" then
			local handle = {closed = false, buffer = ""}
			function handle.write(str)
				if handle.closed then
					return
				end
				handle.buffer = handle.buffer .. str
			end
			function handle.writeLine(str)
				handle.write(str .. "\n")
			end
			function handle.close()
				if handle.closed then
					return
				end
				handle.closed = true
				f.write(handle.buffer)
			end
			return handle
		else
			error("Not supported", 1)
		end
	end
	function p.getSize()
		return 0
	end
	return p
end
function fs.tmpfsProxy(structure, writeCallback, driveName)
	if not structure then
		structure = {files = {}}
	elseif type(structure) == "table" and not structure.files then
		structure.files = {}
	end
	local function getFile(path)
		local file = structure
		for part in string.gmatch(path, "[^/]+") do
			if not file.files[part] then
				error("Not a directory", 1)
			end
			file = file.files[part]
		end
		return file
	end
	local calculateUsed
	function calculateUsed(file)
		return #textutils.serialize(file)
	end
	local p = {}
	function p.import(newstructure)
		structure = newstructure
	end
	function p.dump()
		return structure
	end
	function p.getDrive()
		return driveName or "tmpfs"
	end
	function p.getFreeSpace()
		return -calculateUsed(structure)
	end
	function p.isReadOnly()
		return false
	end
	function p.exists(path)
		local file = structure
		for part in string.gmatch(path, "[^/]+") do
			if not file.files[part] then
				return false
			end
			file = file.files[part]
		end
		return file ~= nil
	end
	function p.isDir(path)
		if path == "" then
			return true
		end
		return p.exists(path) and getFile(path).files ~= nil
	end
	function p.list(path)
		local file
		if path == "" then
			file = structure
		else
			file = getFile(path)
		end
		if not file.files then
			error("Not a directory", 1)
		end
		local files = {}
		for f in pairs(file.files) do
			table.insert(files, f)
		end
		table.sort(files)
		return files
	end
	function p.delete(path)
		if path == "" then
			error("Proxy at path", 1)
		end
		local parent = fs.getDir(path)
		local dir
		if parent == "" then
			dir = structure
		else
			dir = getFile(parent)
		end
		local name = fs.getName(path)
		dir.files[name] = nil
	end
	function p.makeDir(path)
		local parent = structure
		for part in string.gmatch(path, "[^/]+") do
			if not parent.files[part] then
				parent.files[part] = {
					files = {},
				}
			end
			parent = parent.files[part]
		end
	end
	function p.copy(path, target)
		if p.exists(target) then
			error("File exists", 1)
		end
		if not p.exists(path) then
			error("File not found", 1)
		end
		if p.isDir(path) then
			fs.recursiveCopy(p, p, path, target)
		else
			getFile(fs.getDir(target)).files[fs.getName(target)] = {data = getFile(path).data}
		end
	end
	function p.move(path, target)
		getFile(fs.getDir(target)).files[fs.getName(target)] = getFile(fs.getDir(path)).files[fs.getName(path)]
		p.delete(path)
	end
	function p.open(path, mode)
		local handle = {buffer = "", closed = false}
		if mode == "w" or mode == "a" then
			p.makeDir(fs.getDir(path))
			if not p.exists(path) then
				getFile(fs.getDir(path)).files[fs.getName(path)] = {
					data = "",
				}
			end
			handle.file = getFile(path)
			if handle.file.files then
				return
			end
			if mode == "a" then
				handle.buffer = handle.file.data
			end
			function handle.close()
				if handle.closed then return end
				handle.closed = true
				handle.flush()
			end
			function handle.flush()
				handle.file.data = handle.buffer
				if writeCallback then
					writeCallback(structure)
				end
			end
			function handle.write(data)
				if handle.closed then return end
				handle.buffer = handle.buffer .. data
			end
			function handle.writeLine(data)
				handle.write(data .. "\n")
			end
			return handle
		elseif mode == "r" then
			if not p.exists(path) or p.isDir(path) then
				return
			end
			handle.file = getFile(path)
			handle.buffer = handle.file.data
			function handle.readLine()
				if handle.closed then
					return
				end
				if #handle.buffer < 1 then
					return nil
				end
				local tmp = ""
				while #handle.buffer > 0 do
					local chr = handle.buffer:sub(1, 1)
					handle.buffer = handle.buffer:sub(2)
					if chr == "\n" then
						return tmp
					else
						tmp = tmp .. chr
					end
				end
				return tmp
			end
			function handle.readAll()
				if handle.closed then
					return
				end
				local str = handle.buffer
				handle.buffer = ""
				return str
			end
			function handle.close()
				handle.closed = true
			end
			return handle
		else
			error("Mode not supported", 1)
		end
	end
	function p.getSize(path)
		if p.isDir(path) then
			return 0
		end
		return #getFile(path).data
	end
	return p
end
function fs.fileProxy(path, label)
	if not label then
		label = fs.getName(path)
	end
	return fs.tmpfsProxy(aFile.unserializeFile(path) or {}, function(structure)
		aFile.serializeFile(path, structure)
	end, label)
end
function fs.redirectProxy(parent, dir)
	local p = {}
	function p.open(path, mode)
		return parent.open(fs.combine(dir, fs.cleanPath(path)), mode)
	end
	for i, v in ipairs({"delete", "getSize", "getFreeSpace", "makeDir", "getDrive", "isReadOnly", "exists", "isDir", "list"}) do
		p[v] = function(path)
			return parent[v](fs.combine(dir, fs.cleanPath(path)))
		end
	end
	function p.copy(src, dest)
		return parent.copy(fs.combine(dir, fs.cleanPath(src)), fs.combine(dir, fs.cleanPath(dest)))
	end
	function p.move(src, dest)
		return parent.move(fs.combine(dir, fs.cleanPath(src)), fs.combine(dir, fs.cleanPath(dest)))
	end
	return p
end
if not fs.exists("sys") then
	fs.makeDir("sys")
end
if not fs.exists("tmp") then
	fs.makeDir("tmp")
end
local sysStructure = {
	files = {
		reboot = {
			read = nil,
			write = function(data)
				if data == "1" or data == "1\n" then
					os.reboot()
				end
			end,
		},
		shutdown = {
			read = nil,
			write = function(data)
				if data == "1" or data == "1\n" then
					os.shutdown()
				end
			end,
		},
		label = {
			read = function()
				return os.getComputerLabel() or ""
			end,
			write = function(data)
				data = data:gsub("\n", "")
				if data == "" then
					os.setComputerLabel(nil)
				else
					os.setComputerLabel(data)
				end
			end,
		},
	},
}
fs.proxy("sys", fs.appfsProxy(sysStructure))
fs.proxy("tmp", fs.tmpfsProxy())
