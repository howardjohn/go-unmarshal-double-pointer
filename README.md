# Double Pointer Unmarshal Detector

Go programs using `json.Unmarshal` are potentially vulnerable to nil pointer dereferences when used incorrectly.
This subtle mistake is very easy to make, as for almost all inputs - in fact, all but one - the correct code and incorrect code behave the same.

## Example

For example, consider this code snippet:

```go
	cfg := &config{}
	if err := json.Unmarshal(data, &cfg); err != nil {
		return err
	}
	// No errors unmarshalling, must be safe to access...
	fmt.Println(cfg.Data)
```

At first glance, this code looks fine _and behaves fine_. However, it will panic if `data` is `[]byte("null")`.
Since this is a relatively rare input, this issue may go undetected during testing, but expose the application to
behave unexpectedly (likely by crashing, as the above code would when we call `cfg.Data` but `cfg` is `nil`).

[Full Go Playground example](https://go.dev/play/p/V6qSKblMhb3).

## Fixing

Fortunately, the fix is simple: pass a `*config` instead of `**config` to `json.Unmarshal`.

Either of these would fix the above example:

```diff
-	cfg := &config{}
+	cfg := config{}
	if err := json.Unmarshal(data, &cfg); err != nil {
```

```diff
	cfg := &config{}
-	if err := json.Unmarshal(data, &cfg); err != nil {
+	if err := json.Unmarshal(data, cfg); err != nil {
```

## Detecting

This repo provides a tool to detect passing double pointers to `json.Unmarshal`, at runtime.

Usage:
```
# Fetch the code
git clone https://github.com/howardjohn/go-unmarshal-double-pointer /tmp/go-unmarshal-double-pointer
# Run tests against your repository
ALLOWED_DOUBLE_POINTERS=`/tmp/go-unmarshal-double-pointer/known-failures` go test -overlay `/tmp/go-unmarshal-double-pointer/gen-overlay` ./... 
```

This tool works by adding an overlay, replacing the implementation of `json.Unmarshal` with one that panics at runtime when a double pointer is detected as an input.
Sometimes this is done intentionally, or there is a known issue not yet resolved. For these cases, the type can be added to `ALLOWED_DOUBLE_POINTERS`.

The overlay is based on Go 1.17, so is unlikely to work on other versions.
