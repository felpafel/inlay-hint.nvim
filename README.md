# inlay-hint.nvim

This plugin overrides `vim.lsp.inlay_hint` and expose a simple callback that permits the user to edit how inlay hints are displayed, without touching any native API or core logic.

## Demo

![demo](https://github.com/felpafel/inlay-hint.nvim/assets/21080902/7f1c1535-cfb1-4020-bee7-55e6a8a67f4d)

## Motivation

Since inlay hints got integrated to `Neovim` many authors deprecated/archived their own implementations and started calling the `vim.lsp.inlay_hint` api. However, the native API doesn't expose any method to edit how hints are shown in the buffer, and there is an open issue [#28261](https://github.com/neovim/neovim/issues/28261) discussing this topic. So, inspired by the `display_callback` implemented in [nvim-dap-virtual-text](https://github.com/theHamsta/nvim-dap-virtual-text) I decide to create this plugin.

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

> In order to get better completions and type hints inside Neovim, please check [folke/neodev.nvim](https://github.com/folke/neodev.nvim). [completion demo](https://github.com/felpafel/inlay-hint.nvim/assets/21080902/d31ea6b0-dac8-4dca-8ef9-4356d9b2a23d)

```lua
require('inlay-hint').setup({
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

## Credits

**[Neovim team](https://github.com/orgs/neovim/people)**: This plugin is literally a copy and paste of the original `vim.lsp.inlay_hint` implementation.

**[nvim-dap-virtual-text](https://github.com/theHamsta/nvim-dap-virtual-text)**: Who inspired this plugin.
