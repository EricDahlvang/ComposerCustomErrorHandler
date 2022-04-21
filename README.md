# Custom Composer CloudAdapter example

#### **Note: this example uses CosmosDb for storage**

The latest adaptive runtime registers [CoreBotAdapter](https://github.com/microsoft/botbuilder-dotnet/blob/main/libraries/Microsoft.Bot.Builder.Dialogs.Adaptive.Runtime/CoreBotAdapter.cs) as the default adapter in [ServiceCollectionExtensions](https://github.com/microsoft/botbuilder-dotnet/blob/main/libraries/Microsoft.Bot.Builder.Dialogs.Adaptive.Runtime/Extensions/ServiceCollectionExtensions.cs#L101) This contains the default exception handler for the bot. It can be overridden by providing a custom adapter.

# OnErrorAdapter
```cs
using System.Collections.Generic;
using Microsoft.Bot.Builder;
using Microsoft.Bot.Builder.Integration.AspNet.Core;
using Microsoft.Bot.Connector.Authentication;
using Microsoft.Extensions.Logging;

namespace TestError
{
    internal class OnErrorAdapter : CloudAdapter
    {
        public OnErrorAdapter(
            BotFrameworkAuthentication botFrameworkAuthentication,
            IEnumerable<IMiddleware> middlewares,
            ILogger<OnErrorAdapter> logger = null)
            : base(botFrameworkAuthentication, logger)
        {
            // Pick up feature based middlewares such as telemetry or transcripts
            foreach (IMiddleware middleware in middlewares)
            {
                Use(middleware);
            }

            OnTurnError = async (turnContext, exception) =>
            {
                // Log any leaked exception from the application.
                Logger.LogError(exception, $"[OnTurnError] unhandled error : {exception.Message}");

                // Send the exception message to the user. Since the default behavior does not
                // send logs or trace activities, the bot appears hanging without any activity
                // to the user.
                //await turnContext.SendActivityAsync(exception.Message).ConfigureAwait(false);

                await turnContext.SendActivityAsync("Oops, something went wrong. Please start over from the beginning.").ConfigureAwait(false);

                var conversationState = turnContext.TurnState.Get<ConversationState>();

                if (conversationState != null)
                {
                    // Delete the conversationState for the current conversation to prevent the
                    // bot from getting stuck in a error-loop caused by being in a bad state.
                    await conversationState.DeleteAsync(turnContext).ConfigureAwait(false);
                }
            };
        }
    }
}
```

# Startup
```cs
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddControllers().AddNewtonsoftJson();
            services.AddBotRuntime(Configuration);

            services.AddSingleton<OnErrorAdapter>();
            services.AddSingleton<IBotFrameworkHttpAdapter>(sp => sp.GetRequiredService<OnErrorAdapter>());
            services.AddSingleton<BotAdapter>(sp => sp.GetRequiredService<OnErrorAdapter>());
        }
```
# appsettings
```json
  "runtimeSettings": {
    "adapters": [
      {
        "Enabled": true,
        "Route": "messages",
        "Name": "OnErrorAdapter",
        "Type": "TestError.OnErrorAdapter"
      }
    ],
    "features": {
```

# BotController
Finally, comment out the default adapter in `BotController`:
```cs
        public BotController(
            IConfiguration configuration,
            IEnumerable<IBotFrameworkHttpAdapter> adapters,
            IBot bot,
            ILogger<BotController> logger)
        {
            _bot = bot ?? throw new ArgumentNullException(nameof(bot));
            _logger = logger;

            var adapterSettings = configuration.GetSection(AdapterSettings.AdapterSettingsKey).Get<List<AdapterSettings>>() ?? new List<AdapterSettings>();
            //adapterSettings.Add(AdapterSettings.CoreBotAdapterSettings);

            foreach (var adapter in adapters ?? throw new ArgumentNullException(nameof(adapters)))
            {
                var settings = adapterSettings.FirstOrDefault(s => s.Enabled && s.Type == adapter.GetType().FullName);
                
                if (settings != null)
                {
                    _adapters.Add(settings.Route, adapter);
                }
            }
        }
```
