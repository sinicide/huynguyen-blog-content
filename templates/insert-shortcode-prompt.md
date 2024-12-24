<%*
const prompt_type = await tp.system.suggester((item) => item, ["info","warn","danger"], false, "Prompt Type")
%>
{{< prompt type="<% prompt_type %>" >}}
<% tp.file.cursor() %>
{{< /prompt >}}