You're an expert Python developer and architect. You write great code that fulfills the business requirements you get, follows best practices and does not create technological debt.

Business requirement: This app is a chatbot that works on Azure and it works fine. A user asks a question and gets a reply based on vectorized data in AI Search and/or web search and/or Sharepoint search. The app uses Azure AI Search. I would like to add a functionality, so that the chatbot can choose to use an MCP server connection with Microsoft Learn too, apart from the vectorized docs, web search and sharepoint. Queries about Microsoft's technology should be answered based on the Microsoft Learn MCP connection. 

Please note that we shouldn't use key authorization.

You don't have to translate everything into other languages. Just focus on English.

Please assume that the following MCP connection with Microsoft Learn, copied from another app, works:

VERSION 1
#!/usr/bin/env python3
# Semantic Kernel agent with Microsoft Learn MCP server integration

import asyncio
import os
import pathlib
from dotenv import load_dotenv
from semantic_kernel import Kernel
from semantic_kernel.connectors.ai.open_ai import OpenAIChatCompletion ###
from semantic_kernel.connectors.ai.open_ai import AzureChatCompletion  ####
from semantic_kernel.connectors.ai.open_ai import AzureChatPromptExecutionSettings ###

from semantic_kernel.connectors.ai.function_choice_behavior import FunctionChoiceBehavior
from semantic_kernel.connectors.mcp import MCPStreamableHttpPlugin
from semantic_kernel.contents import ChatHistory
from semantic_kernel.contents.streaming_chat_message_content import StreamingChatMessageContent
from semantic_kernel.contents.utils.author_role import AuthorRole
from semantic_kernel.functions import KernelArguments

async def main():
    # Load environment variables from .env file
    current_dir = pathlib.Path(__file__).parent
    env_path = current_dir / ".env"
    load_dotenv(dotenv_path=env_path, override=True)

    print("Setting up the Microsoft Learn Assistant with MCP integration...")
        
    # Initialize the kernel - the orchestration layer for our agent
    kernel = Kernel()
    
    # Add an Azure OpenAI service with function calling enabled
    #api_key = os.getenv("OPENAI_API_KEY")
    api_key = os.getenv("AZURE_OPENAI_API_KEY")
    endpoint = os.getenv("AZURE_OPENAI_ENDPOINT")
    
    if not api_key:
        raise ValueError("AZURE_OPENAI_API_KEY environment variable is not set. Check your .env file.")
    
    print(f"Using API key: {api_key[:5]}...")
    

    model_id = "chat7"  # Choose your model
    service = AzureChatCompletion(deployment_name=model_id, api_key=api_key, endpoint=endpoint)   #orig: OpenAIChatCompletion
    #service = OpenAIChatCompletion(ai_model_id=model_id, api_key=api_key)  
    kernel.add_service(service)
    
    # Create the completion service request settings with function calling enabled
    settings = AzureChatPromptExecutionSettings()
    settings.function_choice_behavior = FunctionChoiceBehavior.Auto()
      # Configure and use the Microsoft Learn MCP plugin via Streamable HTTP
    async with MCPStreamableHttpPlugin(
        name="MicrosoftLearnMCP",
        url="https://learn.microsoft.com/api/mcp"  # Microsoft Learn MCP server endpoint
    ) as mcp_plugin:
        # Register the MCP plugin with the kernel after connecting
        try:
            kernel.add_plugin(mcp_plugin, plugin_name="microsoft_learn")
            print("✅ Successfully connected to Microsoft Learn MCP server")
        except Exception as e:
            print(f"❌ Error: Could not connect to Microsoft Learn MCP server: {str(e)}")
            print("Note: The Microsoft Learn MCP server requires proper authentication and may have usage limits.")
            return
        
        # Create a chat history with system instructions
        history = ChatHistory()
        history.add_system_message(
            "You are a Microsoft Learn assistant with access to Microsoft documentation and learning resources. "
            "Use the available tools to search for and provide information from Microsoft Learn documentation. "
            "You can help users find tutorials, reference materials, code examples, and learning paths."
        )
        
        # Define a simple streaming chat function
        chat_function = kernel.add_function(
            plugin_name="chat",
            function_name="respond", 
            prompt="{{$chat_history}}"
        )
        
        print("\n┌─────────────────────────────────────────────────┐")
        print("│ Microsoft Learn Assistant ready with MCP tools  │")
        print("└─────────────────────────────────────────────────┘")
        print("Type 'exit' to end the conversation.")
        print("\nExample questions:")
        print("- How do I create an Azure Function?")
        print("- Show me tutorials about .NET development")
        print("- What are the best practices for Azure security?")
        print("- Find documentation about Microsoft Graph API")
        
        while True:
            user_input = input("\nUser:> ")
            if user_input.lower() == "exit":
                break
                
            # Add the user message to history
            history.add_user_message(user_input)
            
            # Create streaming arguments with settings for function calling
            arguments = KernelArguments(
                chat_history=history,
                settings=settings
            )
            
            try:
                # Stream the response
                print("Assistant:> ", end="", flush=True)
                
                response_chunks = []
                async for message in kernel.invoke_stream(
                    chat_function,
                    arguments=arguments,
                ):
                    chunk = message[0]
                    if isinstance(chunk, StreamingChatMessageContent) and chunk.role == AuthorRole.ASSISTANT:
                        print(str(chunk), end="", flush=True)
                        response_chunks.append(chunk)
                        
                print()  # New line after response
                
                # Add the full response to history
                full_response = "".join(str(chunk) for chunk in response_chunks)
                history.add_assistant_message(full_response)
                
            except Exception as e:
                print(f"\n❌ Error: {str(e)}")
                print("The Microsoft Learn MCP server may be experiencing issues or rate limiting.")

if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        print("\n\n👋 Exiting the Microsoft Learn Assistant. Goodbye!")
    except ConnectionError:
        print("\n❌ Error: Could not connect to the Microsoft Learn MCP server.")
        print("Please check your internet connection and try again.")
    except Exception as e:
        print(f"\n❌ Error: {str(e)}")
        print("The application has encountered a problem and needs to close.")



VERSION 2

#!/usr/bin/env python3
# Microsoft Agent Framework agent with Microsoft Learn MCP server integration

import asyncio
import os
import pathlib
from dotenv import load_dotenv
from agent_framework import ChatAgent, MCPStreamableHTTPTool
from agent_framework.azure import AzureOpenAIChatClient
from azure.identity.aio import AzureCliCredential

async def main():
    # Load environment variables from .env file
    current_dir = pathlib.Path(__file__).parent
    env_path = current_dir / ".env"
    load_dotenv(dotenv_path=env_path, override=True)

    print("Setting up the Microsoft Learn Assistant with MCP integration...")
    
    # Get Azure OpenAI configuration from environment
    api_key = os.getenv("AZURE_OPENAI_API_KEY")
    endpoint = os.getenv("AZURE_OPENAI_ENDPOINT")
    
    if not api_key or not endpoint:
        raise ValueError("AZURE_OPENAI_API_KEY and AZURE_OPENAI_ENDPOINT environment variables must be set. Check your .env file.")
    
    print(f"Using API key: {api_key[:5]}...")
    
    # Model deployment name
    model_id = "chat7"  # Your Azure OpenAI deployment name
    
    # Create Azure OpenAI Chat Client
    chat_client = AzureOpenAIChatClient(
        endpoint=endpoint,
        api_key=api_key,
        deployment_name=model_id
    )
    
    # Configure the Microsoft Learn MCP tool
    async with (
        MCPStreamableHTTPTool(
            name="Microsoft Learn MCP",
            description="Access to Microsoft documentation and learning resources",
            url="https://learn.microsoft.com/api/mcp"
        ) as mcp_tool,
        ChatAgent(
            chat_client=chat_client,
            name="Microsoft Learn Assistant",
            instructions=(
                "You are a Microsoft Learn assistant with access to Microsoft documentation and learning resources. "
                "Use the available tools to search for and provide information from Microsoft Learn documentation. "
                "You can help users find tutorials, reference materials, code examples, and learning paths."
            ),
            tools=mcp_tool
        ) as agent
    ):
        print("\n✅ Successfully connected to Microsoft Learn MCP server")
        print("\n┌─────────────────────────────────────────────────┐")
        print("│ Microsoft Learn Assistant ready with MCP tools  │")
        print("└─────────────────────────────────────────────────┘")
        print("Type 'exit' to end the conversation.")
        print("\nExample questions:")
        print("- How do I create an Azure Function?")
        print("- Show me tutorials about .NET development")
        print("- What are the best practices for Azure security?")
        print("- Find documentation about Microsoft Graph API")
        
        # Create a thread for multi-turn conversation
        thread = agent.get_new_thread()
        
        while True:
            user_input = input("\nUser:> ")
            if user_input.lower() == "exit":
                break
            
            try:
                # Stream the response
                print("Assistant:> ", end="", flush=True)
                
                async for chunk in agent.run_stream(user_input, thread=thread):
                    if chunk.text:
                        print(chunk.text, end="", flush=True)
                        
                print()  # New line after response
                
            except Exception as e:
                print(f"\n❌ Error: {str(e)}")
                print("The Microsoft Learn MCP server may be experiencing issues or rate limiting.")

if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        print("\n\n👋 Exiting the Microsoft Learn Assistant. Goodbye!")
    except ConnectionError:
        print("\n❌ Error: Could not connect to the Microsoft Learn MCP server.")
        print("Please check your internet connection and try again.")
    except Exception as e:
        print(f"\n❌ Error: {str(e)}")
        print("The application has encountered a problem and needs to close.")