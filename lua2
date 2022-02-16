local builtin = readfile('remap/builtin.json')

local delay_call, userid_to_entindex, type, pairs, ipairs = client.delay_call, client.userid_to_entindex, type, pairs, ipairs
local set_event_callback, unset_event_callback = client.set_event_callback, client.unset_event_callback
local get_local_player, get_mapname, insert, remove = entity.get_local_player, globals.mapname, table.insert, table.remove
local json_parse, matsys_find_material, ui_get, unpack = json.parse, materialsystem.find_material, ui.get, unpack
local sv_skyname = cvar.sv_skyname

local LIMIT = 128
local LIMIT_DELAY = 0.03

local enabled = ui.new_checkbox('VISUALS', 'Effects', 'Remap')
local brightness_adjustment, brightness_adjustment_color = ui.reference('VISUALS', 'Effects', 'Brightness adjustment')

local materials = {}
local enabled_matsheets = {json_parse(builtin)}
local active_matsheet = {fog = {}, bloom = {}, skybox = {}, materials = {}}
local r, g, b = ui_get(brightness_adjustment_color); r, g, b = r/255, g/255, b/255
local adjust_for_nightmode = {
  {key = '$color', explicit = false},
  {key = '$color2', explicit = true},
  {key = '$envmaptint', explicit = true}
}

local delay_call_until_render, clear_delayed, on_post_render

do
  local delayed = {}

  function clear_delayed() delayed = {} end

  function delay_call_until_render(fn, ...)
    insert(delayed, {fn = fn, args = {...}})
  end

  function on_post_render()
    for i = #delayed, 1, -1 do
      delayed[i].fn(unpack(delayed[i].args))
      remove(delayed, i)
    end
  end
end


local function arrcat(a, b)
  for i = 1, #b do
    insert(a, b[i])
  end
end


local function get_map()
  -- TODO: map patterns
  return get_mapname()
end


local function find_material(path)
  if materials[path] ~= nil then return materials[path] end
  materials[path] = matsys_find_material(path)
  return materials[path]
end


local function update_active_matsheet(map)
  active_matsheet = {fog = {}, bloom = {}, skybox = nil, materials = {}}
  for _, ms in ipairs(enabled_matsheets) do
    if ms.globals then
      if ms.globals.materials then
        arrcat(active_matsheet.materials, ms.globals.materials)
      end
    end
    if ms.maps and ms.maps[map] then
      if ms.maps[map].skybox then active_matsheet.skybox = ms.maps[map].skybox end
      if ms.maps[map].materials then
        arrcat(active_matsheet.materials, ms.maps[map].materials)
      end
    end
  end
end


local function get_material_pcount(material_set)
  local pcount, is_multiple = 0
  if material_set.flags then pcount = pcount + #material_set.flags end
  if material_set.params then
    for _ in pairs(material_set.params) do pcount = pcount + 1 end
  end
  if type(material_set.path) == 'table' then
    pcount, is_multiple = pcount * #material_set.path, true
  end
  return pcount, is_multiple
end


local function color_mod(material, params, param, explicit)
  local r_mod, g_mod, b_mod = 1, 1, 1
  if params and params[param] then
    r_mod, g_mod, b_mod = unpack(params[param])
    return material:set_shader_param(param, r * r_mod, g * g_mod, b * b_mod)
  elseif not explicit then
    return material:set_shader_param(param, r * r_mod, g * g_mod, b * b_mod)
  end
end


local function process_params(material, params)
  if not params then return end
  for param, value in pairs(params) do
    if type(value) == 'table' then
      material:set_shader_param(param, unpack(value))
    else
      material:set_shader_param(param, value)
    end
  end
end


local function process_flags(material, flags)
  if not flags then return end
  for _, flag in ipairs(flags) do
    material:set_material_var_flag(flag[1], flag[2])
  end
end


local function process_material(path, material_set, do_color_mod)
  local material = find_material(path)
  if not material then return end
  process_params(material, material_set.params)
  process_flags(material, material_set.flags)
  if do_color_mod then
    for _, param in ipairs(adjust_for_nightmode) do
      color_mod(material, material_set.params, param.key, param.explicit)
    end
    if material_set.nightmode then
      process_params(material, material_set.nightmode)
    end
  end
end


local function process_materials(material_set, do_color_mod)
  local paths = material_set.path
  for i = 1, #paths do
    process_material(paths[i], material_set, do_color_mod)
  end
end


local function process_matsheet(ms, lim, idx)
  local is_nightmode = ui_get(brightness_adjustment) == 'Night mode'
  lim, idx = lim or LIMIT, idx or 1

  local pcount = 0
  for i = idx, #ms.materials do
    local material_set = ms.materials[i]
    local this_pcount, is_multiple = get_material_pcount(material_set)
    if this_pcount+pcount <= lim or this_pcount > lim then
      if is_multiple then
        process_materials(material_set, is_nightmode)
      else
        process_material(material_set.path, material_set, is_nightmode)
      end
      pcount = pcount + this_pcount
      if this_pcount > lim and i < #ms.materials then
        return delay_call(LIMIT_DELAY, process_matsheet, ms, lim, i+1)
      end
    else
      return delay_call(LIMIT_DELAY, process_matsheet, ms, lim, i)
    end
  end
end


local function process_matsheet_colors(ms)
  for i = 1, #ms.materials do
    local material_set = ms.materials[i]
    local paths = material_set.path
    if type(paths) == 'string' then paths = {paths} end
    for _, path in ipairs(paths) do
      local material = find_material(path)
      for _, param in ipairs(adjust_for_nightmode) do
        color_mod(material, material_set.params, param.key, true)
      end
      if material_set.nightmode then
        process_params(material, material_set.nightmode)
      end
    end
  end
end


local function on_enable()
  materials = {}
  local map = get_map()
  update_active_matsheet(map)
  process_matsheet(active_matsheet, LIMIT)
end


local function on_disable()
  if active_matsheet.skybox then
    if active_matsheet.skybox.skyname and active_matsheet.skybox.restore_skyname then
      sv_skyname:set_string(active_matsheet.skybox.restore_skyname)
    end
  end
  -- TODO: Bloom
  -- TODO: Fog
  local is_nightmode = ui_get(brightness_adjustment) == 'Night mode'
  for _, material in pairs(materials) do
    material:reload()
    if is_nightmode then
      material:set_shader_param('$color', r, g, b)
      -- material:set_shader_param('$envmaptint', r, g, b)
    end
  end
end


local function on_player_connect_full(e)
  if userid_to_entindex(e.userid) == get_local_player() then
    delay_call(2, on_enable)
  end
end


ui.set_callback(enabled, function(self)
  if ui_get(self) then
    set_event_callback('player_connect_full', on_player_connect_full)
    set_event_callback('post_render', on_post_render)
    on_enable()
  else
    unset_event_callback('player_connect_full', on_player_connect_full)
    unset_event_callback('post_render', on_post_render)
    clear_delayed()
    on_disable()
  end
end)


ui.set_callback(brightness_adjustment, function(self)
  if not ui_get(enabled) then return end
  if ui_get(self) == 'Off' then
    delay_call_until_render(process_matsheet, active_matsheet)
  elseif ui_get(self) == 'Night mode' then
    delay_call_until_render(process_matsheet_colors, active_matsheet)
  end
end)


ui.set_callback(brightness_adjustment_color, function(self)
  local _r, _g, _b = ui_get(self)
  r, g, b = _r/255, _g/255, _b/255
  if ui_get(enabled) and ui_get(brightness_adjustment) == 'Night mode' then
    delay_call_until_render(process_matsheet_colors, active_matsheet)
  end
end)
