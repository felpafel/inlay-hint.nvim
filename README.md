# inlay-hint.nvim

This plugin overrides `vim.lsp.inlay_hint` and expose a simple callback that permits the user to edit how inlay hints are displayed, without changing any native API or core logic.

## Demo

- EOL

![eol](https://github.com/user-attachments/assets/ef6afaa4-de6f-44a4-af9a-87ca50ca5b6d)

- Inline

![inline](https://github.com/user-attachments/assets/98e2a4be-202a-4d0c-9f3c-36bac7fb4665)

- Right Align

![right_align](https://github.com/user-attachments/assets/b55edf35-7f97-46d0-947e-b7fa471c2fb7)

<details>
  <summary>
    Lua
  </summary>

Default

![lua-default](https://github.com/user-attachments/assets/71d19c26-6f26-43a8-97e6-c8d26f2c687c)

[Treesitter](#using-treesitter-to-show-variables-and-their-types)

![lua-treesitter](https://github.com/user-attachments/assets/ed39b319-97c2-4f17-a890-9b9cb3070471)

</details>

<details>
  <summary>
    Rust
  </summary>

Default

![rust-default](https://github.com/user-attachments/assets/cc488ad4-11a8-44cb-bb58-549e17f4ad2d)

[Treesitter](#using-treesitter-to-show-variables-and-their-types)

![rust-treesitter](https://github.com/user-attachments/assets/0972764c-d14e-4c2c-bce2-a416718b1265)

</details>
<details>
  <summary>
    Zig
  </summary>

Default

![zig-default](https://github.com/user-attachments/assets/83419aa3-1c5c-4068-9073-fd13ba6e2725)

[Treesitter](#using-treesitter-to-show-variables-and-their-types)

![zig-treesitter](https://github.com/user-attachments/assets/45878a6c-647b-408e-8e5c-d073c5a621d8)

</details>

I limited the demo to 3 languages, but this plugin works _out of the box_ with any LSP that implement inlay hints.

## Motivation

Since inlay hints got integrated to `Neovim` many authors deprecated/archived their own implementations and started calling the `vim.lsp.inlay_hint` api. However, the native API doesn't expose any method to edit how hints are shown in the buffer (see issue [#28261](https://github.com/neovim/neovim/issues/28261) for more details on this topic). So, inspired by the `display_callback` implemented in [nvim-dap-virtual-text](https://github.com/theHamsta/nvim-dap-virtual-text) I decide to create this plugin.

## Prerequisites

- [`NVIM v0.10.0`](https://github.com/neovim/neovim/releases/tag/v0.10.0)

## Installation

- With [folke/lazy.nvim](https://github.com/folke/lazy.nvim)

```lua
{
  'felpafel/inlay-hint.nvim',
  event = 'LspAttach',
  config = true,
}
```

## Configuration

The behavior of this plugin can be activated and controlled via a `setup` call.

```lua
require('inlay-hint').setup()
```

<details>
  <summary>
	Or with additional options
  </summary>

> In order to get better completions and type hints inside Neovim, please check [folke/lazydev.nvim](https://github.com/folke/lazydev.nvim). [completion demo](https://github.com/felpafel/inlay-hint.nvim/assets/21080902/6cf9c785-0cb7-43fc-9d40-f1f9c0f6e0fc)

`virt_text_pos`, `highlight_group` and `hl_mode` are the same options present in `nvim_buf_set_extmark()`

```lua
require('inlay-hint').setup({
  -- Position of virtual text. Possible values:
  -- 'eol': right after eol character (default).
  -- 'right_align': display right aligned in the window.
  -- 'inline': display at the specified column, and shift the buffer
  -- text to the right as needed.
  virt_text_pos = 'eol',
  -- Can be supplied either as a string or as an integer,
  -- the latter which can be obtained using |nvim_get_hl_id_by_name()|.
  highlight_group = 'LspInlayHint',
  -- Control how highlights are combined with the
  -- highlights of the text.
  -- 'combine': combine with background text color. (default)
  -- 'replace': only show the virt_text color.
  hl_mode = 'combine',
  -- line_hints: array with all hints present in current line.
  -- options: table with this plugin configuration.
  -- bufnr: buffer id from where the hints come from.
  display_callback = function(line_hints, options, bufnr, winid)
    if options.virt_text_pos == 'inline' then
      local lhint = {}
      for _, hint in pairs(line_hints) do
        local text = ''
        local label = hint.label
        if type(label) == 'string' then
          text = label
        else
          for _, part in ipairs(label) do
            text = text .. part.value
          end
        end
        if hint.paddingLeft then
          text = ' ' .. text
        end
        if hint.paddingRight then
          text = text .. ' '
        end
        lhint[#lhint + 1] = { text = text, col = hint.position.character }
      end
      return lhint
    elseif options.virt_text_pos == 'eol' or options.virt_text_pos == 'right_align' then
      local k1 = {}
      local k2 = {}
      table.sort(line_hints, function(a, b)
        return a.position.character < b.position.character
      end)
      for _, hint in pairs(line_hints) do
        local label = hint.label
        local kind = hint.kind
        local text = ''
        if type(label) == 'string' then
          text = label
        else
          for _, part in ipairs(label) do
            text = text .. part.value
          end
        end
        if kind == 1 then
          k1[#k1 + 1] = text:gsub('^:%s*', '')
        else
          k2[#k2 + 1] = text:gsub(':$', '')
        end
      end
      local text = ''
      if #k2 > 0 then
        text = '<- (' .. table.concat(k2, ',') .. ')'
      end
      if #text > 0 then
        text = text .. ' '
      end
      if #k1 > 0 then
        text = text .. '=> ' .. table.concat(k1, ',')
      end

      return text
    end
    return nil
  end,
})
```

</details>

<details>
  <summary>
      Enable hints on LspAttach and toggle keymap
  </summary>

```lua
vim.api.nvim_create_autocmd('LspAttach', {
  callback = function(args)
    local bufnr = args.buf ---@type number
    local client = vim.lsp.get_client_by_id(args.data.client_id)
    if client.supports_method('textDocument/inlayHint') then
      vim.lsp.inlay_hint.enable(true, { bufnr = bufnr })
      vim.keymap.set('n', '<leader>i', function()
        vim.lsp.inlay_hint.enable(
          not vim.lsp.inlay_hint.is_enabled({ bufnr = bufnr }),
          { bufnr = bufnr }
        )
      end, { buffer = bufnr })
    end
  end,
})
```

</details>

## Using Treesitter to show variables and their types

```lua
require('inlay-hint').setup({
  display_callback = function(line_hints, options, bufnr, winid)
  if options.virt_text_pos == 'inline' then
    local lhint = {}
    for _, hint in pairs(line_hints) do
      local text = ''
      local label = hint.label
      if type(label) == 'string' then
        text = label
      else
        for _, part in ipairs(label) do
          text = text .. part.value
        end
      end
      if hint.paddingLeft then
        text = ' ' .. text
      end
      if hint.paddingRight then
        text = text .. ' '
      end
      lhint[#lhint + 1] = { text = text, col = hint.position.character }
    end
    return lhint
  elseif options.virt_text_pos == 'eol' or options.virt_text_pos == 'right_align' then
    local k1 = {}
    local k2 = {}
    table.sort(line_hints, function(a, b)
      return a.position.character < b.position.character
    end)
    for _, hint in pairs(line_hints) do
      local label = hint.label
      local kind = hint.kind
      local node = kind == 1
      and vim.treesitter.get_node({
        bufnr = bufnr,
        pos = {
          hint.position.line,
          hint.position.character - 1,
        },
      })
      or nil
      local node_text = node and vim.treesitter.get_node_text(node, bufnr, {}) or ''
      local text = ''
      if type(label) == 'string' then
        text = label
      else
        for _, part in ipairs(label) do
          text = text .. part.value
        end
      end
      if kind == 1 then
        k1[#k1 + 1] = text:gsub(':%s*', node_text .. ': ')
      else
        k2[#k2 + 1] = text:gsub(':$', '')
      end
    end
    local text = ''
    if #k2 > 0 then
      text = '<- (' .. table.concat(k2, ',') .. ')'
    end
    if #text > 0 then
      text = text .. ' '
    end
    if #k1 > 0 then
      text = text .. '=> ' .. table.concat(k1, ', ')
    end

    return text
  end
  return nil
end,
})
```

## Credits

**[Neovim team](https://github.com/orgs/neovim/people)**: This plugin is literally a copy and paste of the original `vim.lsp.inlay_hint` implementation.

**[nvim-dap-virtual-text](https://github.com/theHamsta/nvim-dap-virtual-text)**: Who inspired this plugin.
