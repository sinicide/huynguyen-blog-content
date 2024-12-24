<%*
const image_path = await tp.system.prompt("Enter Image")
const image_alt = await tp.system.prompt("Enter Image alt")
const image_capation = await tp.system.prompt("Enter Image Caption")
%>{{< figure src="<% image_path %>" alt="<% image_alt %>" caption="<% image_caption %>" >}}