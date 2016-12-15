---
title:   "Setting the default button for a nested control in ASP.NET"
date:    2008-06-24 11:55:00 UTC
---

In ASP.NET 2.0, you can declaratively set which button which be "clicked" when the user presses the Enter key. This can be done on either the `<form>` or `<asp:Panel>` tag, as follows:

``` xml
<asp:Panel runat="server" DefaultButton="btnSubmit">
	<asp:TextBox runat="server" ID="txtFirstName" />
	<asp:Button runat="server" ID="btnSubmit" Text="Submit" />
</asp:Panel>
```

This works fine as long as the button control is directly contained within the Panel or form. However, if the button control is nested within a user control or any other control that implements INamingContainer, it will no longer work.

I discovered that you can still do this declaratively - but you need to use the *relative* ID. For the sake of argument, let's say that the button is contained within a CreateUserWizard control (this could be any control or user control though).

For this scenario, you'd need to set the DefaultButton property to the path *relative to the Panel on which you're setting the DefaultButton*. For example (note that "__CustomNav0" might need to be replaced - I get this ID because of a custom adapter I use on the CreateUserWizard control):

``` xml
<asp:Panel runat="server" DefaultButton="cuwRegister$__CustomNav0$btnSubmit">
	<asp:CreateUserWizard runat="server" ID="cuwRegister">
		<WizardSteps>
			<asp:CompleteWizardStep runat="server">
				<CustomNavigationTemplate>
					<asp:Button runat="server" ID="btnSubmit" Text="Submit" />
				</CustomNavigationTemplate>
			</asp:CompleteWizardStep>
		</WizardSteps>
	</asp:CreateUserWizard>
</asp:Panel>
```

I hope this helps somebody.