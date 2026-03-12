# Workshop: Building a Code-Editing Agent with Go and Anthropic


Pre-requisites
- install go https://go.dev/
- get an Anthropic API key


Adapted from
https://ampcode.com/notes/how-to-build-an-agent


You can build an agent in less than 400 lines of code, most of which is boiler plate.

Following along allows you to feel how little code it is and I want you to see this with your own eyes in your own terminal in your own folders.


## 0. Pre-requisites
- install go https://go.dev/
- get an Anthropic API key
(get a back-up key as a spare)


## 1. Setup

Create a new Go project:

```bash
mkdir code-editing-agent
cd code-editing-agent
go mod init agent
touch main.go
```

## 2. Skeleton

Open `main.go` and add the basic structure. This sets up the Anthropic client and a way to read user input.

```go
package main

import (
    "bufio"
    "context"
    "fmt"
    "os"
    "github.com/anthropics/anthropic-sdk-go"
)

type Agent struct {
    client         *anthropic.Client
    getUserMessage func() (string, bool)
}

func main() {
    client := anthropic.NewClient()
    scanner := bufio.NewScanner(os.Stdin)

    getUserMessage := func() (string, bool) {
        if !scanner.Scan() {
            return "", false
        }
        return scanner.Text(), true
    }

    agent := NewAgent(&client, getUserMessage)
    err := agent.Run(context.TODO())
    if err != nil {
        fmt.Printf("Error: %s\n", err.Error())
    }
}

func NewAgent(client *anthropic.Client, getUserMessage func() (string, bool)) *Agent {
    return &Agent{
        client: client,
        getUserMessage: getUserMessage,
    }
}
```




## 3. The Heartbeat (The Loop)
The core of an agent is a loop that maintains conversation state and interacts with the LLM.

```go
func (a *Agent) Run(ctx context.Context) error {
    conversation := []anthropic.MessageParam{}
    fmt.Println("Chat with Claude (use 'ctrl-c' to quit)")

    for {
        fmt.Print("\u001b[94mYou\u001b[0m: ")
        userInput, ok := a.getUserMessage()
        if !ok { break }

        userMessage := anthropic.NewUserMessage(anthropic.NewTextBlock(userInput))
        conversation = append(conversation, userMessage)

        message, err := a.runInference(ctx, conversation)
        if err != nil { return err }

        conversation = append(conversation, message.ToParam())

        for _, content := range message.Content {
            if content.Type == "text" {
                fmt.Printf("\u001b[93mClaude\u001b[0m: %s\n", content.Text)
            }
        }
    }
    return nil
}

func (a *Agent) runInference(ctx context.Context, conversation []anthropic.MessageParam) (*anthropic.Message, error) {
    return a.client.Messages.New(ctx, anthropic.MessageNewParams{
        Model:     anthropic.ModelClaude4Sonnet20250514,
        MaxTokens: int64(1024),
        Messages:  conversation,
    })
}
```


```bash
export ANTHROPIC_API_KEY="this is the last time i will tell you to set this"

# Download the dependencies
go mod tidy
# Run it
go run main.go
```

Let's test it! 
Ask Claude what it can do. Notice that it doesn't have any tools yet, so it will just say it can chat and answer questions.



## 4. Turning the Chatbot into an Agent (Tools)
An agent is an LLM with access to tools—the ability to modify something outside the context window.

Defining a Tool
Each tool needs:
- Name
- Description
- Input Schema (JSON schema)
- Function (The actual Go code to execute)


So let’s add that to our code:

```go
// main.go

type ToolDefinition struct {
	Name        string                         `json:"name"`
	Description string                         `json:"description"`
	InputSchema anthropic.ToolInputSchemaParam `json:"input_schema"`
	Function    func(input json.RawMessage) (string, error)
}
```

Now we give our Agent tool definitions:

```go
// main.go

// `tools` is added here:
type Agent struct {
	client         *anthropic.Client
	getUserMessage func() (string, bool)
	tools          []ToolDefinition
}

// And here:
func NewAgent(
	client *anthropic.Client,
	getUserMessage func() (string, bool),
	tools []ToolDefinition,
) *Agent {
	return &Agent{
		client:         client,
		getUserMessage: getUserMessage,
		tools:          tools,
	}
}

// And here:
func main() {
    // [... previous code ...]
	tools := []ToolDefinition{}
	agent := NewAgent(&client, getUserMessage, 	tools)

    // [... previous code ...]
}
```

And send them along to the model in runInference:

```go
// main.go

func (a *Agent) runInference(ctx context.Context, conversation []anthropic.MessageParam) (*anthropic.Message, error) {
	anthropicTools := []anthropic.ToolUnionParam{}
	for _, tool := range a.tools {
		anthropicTools = append(anthropicTools, anthropic.ToolUnionParam{
			OfTool: &anthropic.ToolParam{
				Name:        tool.Name,
				Description: anthropic.String(tool.Description),
				InputSchema: tool.InputSchema,
			},
		})
	}

	message, err := a.client.Messages.New(ctx, anthropic.MessageNewParams{
		Model:     anthropic.ModelClaude4Sonnet20250514,
		MaxTokens: int64(1024),
		Messages:  conversation,
		Tools:     anthropicTools,
	})
	return message, err
}
```

## 5. Handling Tool Use
When Claude requests a tool (winks), the program must:

1. Intercept the tool request.
2. Execute the local Go function.
3. Send the results back to Claude.


Example: `read_file`
The implementation of a simple file-reading tool:

Alright, so tool definitions are being sent along, but we haven’t defined a tool yet. Let’s do that and define read_file:

```go
// main.go

var ReadFileDefinition = ToolDefinition{
	Name:        "read_file",
	Description: "Read the contents of a given relative file path. Use this when you want to see what's inside a file. Do not use this with directory names.",
	InputSchema: ReadFileInputSchema,
	Function:    ReadFile,
}

type ReadFileInput struct {
	Path string `json:"path" jsonschema_description:"The relative path of a file in the working directory."`
}

var ReadFileInputSchema = GenerateSchema[ReadFileInput]()

func ReadFile(input json.RawMessage) (string, error) {
	readFileInput := ReadFileInput{}
	err := json.Unmarshal(input, &readFileInput)
	if err != nil {
		panic(err)
	}

	content, err := os.ReadFile(readFileInput.Path)
	if err != nil {
		return "", err
	}
	return string(content), nil
}

func GenerateSchema[T any]() anthropic.ToolInputSchemaParam {
	reflector := jsonschema.Reflector{
		AllowAdditionalProperties: false,
		DoNotReference:            true,
	}
	var v T

	schema := reflector.Reflect(v)

	return anthropic.ToolInputSchemaParam{
		Properties: schema.Properties,
	}
}
```

That’s not much, is it? It’s a single function, ReadFile, and two descriptions the model will see: our Description that describes the tool itself ("Read the contents of a given relative file path. ...") and a description of the single input parameter this tool has ("The relative path of a ...").

The ReadFileInputSchema and GenerateSchema stuff? We need that so that we can generate a JSON schema for our tool definition which we send to the model. To do that, we use the jsonschema package, which we need to import and download:

```go
// main.go

package main

import (
	"bufio"
	"context"
    // Add this:
	"encoding/json"
	"fmt"
	"os"

	"github.com/anthropics/anthropic-sdk-go"
    // Add this:
	"github.com/invopop/jsonschema"
)
```

Then run the following:

```bash
go mod tidy
```


Then, in the main function, we need to make sure that we use the definition:

```go
func main() {
    // [... previous code ...]
	tools := []ToolDefinition{ReadFileDefinition}
    // [... previous code ...]
}
```


Time to try it!
 


Replace our Agent’s Run method with this:

```go
// main.go

func (a *Agent) Run(ctx context.Context) error {
	conversation := []anthropic.MessageParam{}

	fmt.Println("Chat with Claude (use 'ctrl-c' to quit)")

	readUserInput := true
	for {
		if readUserInput {
			fmt.Print("\\u001b[94mYou\\u001b[0m: ")
			userInput, ok := a.getUserMessage()
			if !ok {
				break
			}

			userMessage := anthropic.NewUserMessage(anthropic.NewTextBlock(userInput))
			conversation = append(conversation, userMessage)
		}

		message, err := a.runInference(ctx, conversation)
		if err != nil {
			return err
		}
		conversation = append(conversation, message.ToParam())

		toolResults := []anthropic.ContentBlockParamUnion{}
		for _, content := range message.Content {
			switch content.Type {
			case "text":
				fmt.Printf("\\u001b[93mClaude\\u001b[0m: %s\n", content.Text)
			case "tool_use":
				result := a.executeTool(content.ID, content.Name, content.Input)
				toolResults = append(toolResults, result)
			}
		}
		if len(toolResults) == 0 {
			readUserInput = true
			continue
		}
		readUserInput = false
		conversation = append(conversation, anthropic.NewUserMessage(toolResults...))
	}

	return nil
}

func (a *Agent) executeTool(id, name string, input json.RawMessage) anthropic.ContentBlockParamUnion {
	var toolDef ToolDefinition
	var found bool
	for _, tool := range a.tools {
		if tool.Name == name {
			toolDef = tool
			found = true
			break
		}
	}
	if !found {
		return anthropic.NewToolResultBlock(id, "tool not found", true)
	}

	fmt.Printf("\\u001b[92mtool\\u001b[0m: %s(%s)\n", name, input)
	response, err := toolDef.Function(input)
	if err != nil {
		return anthropic.NewToolResultBlock(id, err.Error(), true)
	}
	return anthropic.NewToolResultBlock(id, response, false)
}
```

To test:
- ask claude to solve the riddle in the in the file `riddle.txt`
- ask Claude to explain the content of `main.go`



## 6. List file tool

Let's add the ability to List files

```go
// main.go

var ListFilesDefinition = ToolDefinition{
	Name:        "list_files",
	Description: "List files and directories at a given path. If no path is provided, lists files in the current directory.",
	InputSchema: ListFilesInputSchema,
	Function:    ListFiles,
}

type ListFilesInput struct {
	Path string `json:"path,omitempty" jsonschema_description:"Optional relative path to list files from. Defaults to current directory if not provided."`
}

var ListFilesInputSchema = GenerateSchema[ListFilesInput]()

func ListFiles(input json.RawMessage) (string, error) {
	listFilesInput := ListFilesInput{}
	err := json.Unmarshal(input, &listFilesInput)
	if err != nil {
		panic(err)
	}

	dir := "."
	if listFilesInput.Path != "" {
		dir = listFilesInput.Path
	}

	var files []string
	err = filepath.Walk(dir, func(path string, info os.FileInfo, err error) error {
		if err != nil {
			return err
		}

		relPath, err := filepath.Rel(dir, path)
		if err != nil {
			return err
		}

		if relPath != "." {
			if info.IsDir() {
				files = append(files, relPath+"/")
			} else {
				files = append(files, relPath)
			}
		}
		return nil
	})

	if err != nil {
		return "", err
	}

	result, err := json.Marshal(files)
	if err != nil {
		return "", err
	}

	return string(result), nil
}
```


Tell Claude about `list_files` too:

```go
// main.go

func main() {
	// [... previous code ...]

	tools := []ToolDefinition{ReadFileDefinition, ListFilesDefinition}

	// [... previous code ...]
}
```

To test: 
- ask Claude to list files in the current directory, and then ask it to read one of the files. You can also ask it to list files in a subdirectory if you have one.
- ask it to list files in a non-existent directory to see the error handling.
- Combining tools, test with "tell me ab out go files, be brief"
- Ask what go version are we using


## 7. Add edit files

First let's add the definition:

```go
// main.go

var EditFileDefinition = ToolDefinition{
	Name: "edit_file",
	Description: `Make edits to a text file.

Replaces 'old_str' with 'new_str' in the given file. 'old_str' and 'new_str' MUST be different from each other.

If the file specified with path doesn't exist, it will be created.
`,
	InputSchema: EditFileInputSchema,
	Function:    EditFile,
}

type EditFileInput struct {
	Path   string `json:"path" jsonschema_description:"The path to the file"`
	OldStr string `json:"old_str" jsonschema_description:"Text to search for - must match exactly and must only have one match exactly"`
	NewStr string `json:"new_str" jsonschema_description:"Text to replace old_str with"`
}

var EditFileInputSchema = GenerateSchema[EditFileInput]()
```

Now here’s the implementation of the EditFile function in Go:

```go
func EditFile(input json.RawMessage) (string, error) {
	editFileInput := EditFileInput{}
	err := json.Unmarshal(input, &editFileInput)
	if err != nil {
		return "", err
	}

	if editFileInput.Path == "" || editFileInput.OldStr == editFileInput.NewStr {
		return "", fmt.Errorf("invalid input parameters")
	}

	content, err := os.ReadFile(editFileInput.Path)
	if err != nil {
		if os.IsNotExist(err) && editFileInput.OldStr == "" {
			return createNewFile(editFileInput.Path, editFileInput.NewStr)
		}
		return "", err
	}

	oldContent := string(content)
	newContent := strings.Replace(oldContent, editFileInput.OldStr, editFileInput.NewStr, -1)

	if oldContent == newContent && editFileInput.OldStr != "" {
		return "", fmt.Errorf("old_str not found in file")
	}

	err = os.WriteFile(editFileInput.Path, []byte(newContent), 0644)
	if err != nil {
		return "", err
	}

	return "OK", nil
}
```

It checks the input parameters, it reads the file (or creates it if it exists), and replaces the OldStr with NewStr. Then it writes the content back to disk and returns "OK".


What’s missing still is createNewFile:

```go
func createNewFile(filePath, content string) (string, error) {
	dir := path.Dir(filePath)
	if dir != "." {
		err := os.MkdirAll(dir, 0755)
		if err != nil {
			return "", fmt.Errorf("failed to create directory: %w", err)
		}
	}

	err := os.WriteFile(filePath, []byte(content), 0644)
	if err != nil {
		return "", fmt.Errorf("failed to create file: %w", err)
	}

	return fmt.Sprintf("Successfully created file %s", filePath), nil
}
```

Last step: adding it to the list of tools that we send to Claude.

```go
// main.go

func main() {
	// [... previous code ...]

	tools := []ToolDefinition{ReadFileDefinition, ListFilesDefinition, EditFileDefinition}

	// [... previous code ...]
}
```

Run an test this by asking Claude to create a new file, list files to see it, and then edit it.




## Isn’t this amazing?
<quote>
If you’re anything like all the engineers I’ve talked to in the past few months, chances are that, while reading this, you have been waiting for the rabbit to be pulled out of the hat, for me to say “well, in reality it’s much, much harder than this.” But it’s not.

This is essentially all there is to the inner loop of a code-editing agent. Sure, integrating it into your editor, tweaking the system prompt, giving it the right feedback at the right time, a nice UI around it, better tooling around the tools, support for multiple agents, and so on — we’ve built all of that in Amp, but it didn’t require moments of genius. All that was required was practical engineering and elbow grease.

These models are incredibly powerful now. 300 lines of code and three tools and now you’re to be able to talk to an alien intelligence that edits your code. If you think “well, but we didn’t really…” — go and try it! Go and see how far you can get with this. I bet it’s a lot farther than you think.
</quote>
https://ampcode.com/notes/how-to-build-an-agent


## 8. Next Steps


Use AI assisted coding to
- add a tool that can run tests and return results
- add more tools that you think would be useful
- add guardrails (e.g. don't allow deleting code, only editing existing code, or only allow editing code between certain comments like // BEGIN EDIT and // END EDIT)
- add a tool


Look at improving the agents files
- ask the AI coding assistant to generate one, consider adding Dave Cheny's best practices and asking the assistant to follow them when generating code for the agent file and tell which rules are being applied in the generation
- https://github.com/willvelida/code-minions