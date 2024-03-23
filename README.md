# inlay-hint.nvim

This plugin overrides `vim.lsp.inlay_hint` and expose a simple callback that permits the user to edit how inlay hints are displayed, without touching any native API or core logic.

## Demo

https://github.com/felpafel/inlay-hint.nvim/assets/21080902/ac689452-5f11-42b5-9c6e-53960d48145d

## Motivation

Since inlay hints got integrated to `neovim nightly` many authors deprecated/archived their own implementations and started using the `vim.lsp.inlay_hint` api. However, the native API doesn't expose any method to edit how hints are shown in the buffer. So, inspired by the `display_callback` implemented in [nvim-dap-virtual-text](https://github.com/theHamsta/nvim-dap-virtual-text) I decide to create this plugin.

## Prerequisites

- `neovim v0.10.0` with `vim.lsp.inlay_hint` API.

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

> In order get better completions and type hints inside Neovim, please check [folke/neodev.nvim](https://github.com/folke/neodev.nvim).

````lua
require('inlay-hint').setup({
  --- If `override_native_inlay_hint` is set to `false`, you have to manually
  --- attach to lsp-handlers:
  ---
  --- ```lua
  --- local inlay_hint = require('inlay-hint')
  --- inlay_hint.setup({ override_native_inlay_hint = false })
  --- lsp.handler['workspace/inlayHint/refresh'] = function(err, result, ctx, config)
  ---   return inlay_hint.on_refresh(err, result, ctx, config)
  --- end
  --- lsp.handler['textDocument/inlayHint'] = function(...)
  ---   return inlay_hint.on_inlayhint(...)
  --- end
  --- ```
  override_native_inlay_hint = true,
  virt_text_pos = 'eol',
  highlight_group = 'LspInlayHint',
  hl_mode = 'combine',
  display_callback = function(line_hints, options)
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
        lhint[#lhint + 1] =
        { text = text, col = hint.position.character }
      end
      return lhint
    elseif
      options.virt_text_pos == 'eol'
      or options.virt_text_pos == 'right_align'
    then
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
      if #k1 > 0 then
        text = '=> ' .. table.concat(k1, ',')
      end
      if #k2 > 0 then
        text = '<- (' .. table.concat(k2, ',') .. ')'
      end
      return text
    end
    return nil
  end,
})
````

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
    vim.lsp.inlay_hint.enable(bufnr, true)
    vim.keymap.set('n', '<leader>uh', function()
      vim.lsp.inlay_hint.enable(
        bufnr,
        not vim.lsp.inlay_hint.is_enabled(bufnr)
      )
    end, { buffer = bufnr })
  end
end,
})
```

</details>

## Credits

**[Neovim team](https://github.com/orgs/neovim/people)**: This plugin is literally a copy and paste of the original `vim.lsp.inlay_hint` implementation.

**[nvim-dap-virtual-text](https://github.com/theHamsta/nvim-dap-virtual-text)**: Who inspired this plugin.
