<%*
const image_path = await tp.system.prompt("Enter Image")
const image_alt = await tp.system.prompt("Enter Image alt")
%>{{< image src="<% image_path %>" alt="<% image_alt %>" >}}