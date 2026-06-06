 let's start a new project. The goal is to create an MCP server that allows me to say in Claude 'send that markdown file to my kindle'. It should be an MCP server written in typescript that Claude Code can use. We'll need to implement a new endpoint in @../readitsoon/ that accepts pure markdown, a kindle e-mail and
  sends the markdown as an epub to the kindle email. The title of the epub is the name of the markdown file, the author will be the user.
  To authenticate the MCP readitsoon will need to implement a new OAUTH 2.1 flow for the MCP to follow.
  Let's make a plan.
