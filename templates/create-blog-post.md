<%*
const blog_title = await tp.system.prompt("Blog Title")

// replace spaces with "-"
let blog_post_path = "/blog/" + tp.file.creation_date("YYYY/MM/DD/") + blog_title.replace(/\ /g, "-") + "/index"

// move file to correct path
try {
	await tp.file.move(blog_post_path)
} catch (err) {
	console.error(`${err}`);
}
%><% "---" %>
layout: blog
title: "<% blog_title %>"
date: <% tp.file.creation_date("YYYY-MM-DDTHH:mm:ssZ") %>
lastMod: <% tp.file.last_modified_date("YYYY-MM-DDTHH:mm:ssZ") %>
categories: 
tags: 
description: 
disableComments: true
draft: true
<% "---" %>
<% tp.file.cursor() %>